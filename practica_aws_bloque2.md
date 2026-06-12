# Práctica AWS — Tienda Master UCM
## Bloque 2: Cómputo y exposición (ALB, ECS Fargate y API Gateway)

> **Punto de partida:** Bloque 1 completado (variables, default VPC, *security groups*, RDS y la imagen del backend en ECR).
> **En este bloque** levantamos un **ALB interno**, desplegamos el backend en **ECS Fargate** (inyectando las credenciales de RDS desde Secrets Manager) y lo publicamos en Internet a través de **API Gateway + VPC Link**.

Al empezar, recarga siempre las variables:

```bash
source setup-env.sh
```

---

## Fase 5 — Balanceador de carga interno (ALB)

El backend correrá en varias tareas Fargate con IPs cambiantes. El ALB les da un único punto de entrada estable y reparte el tráfico. Lo creamos **interno** (`--scheme internal`): no tendrá IP pública porque quien le hablará es API Gateway desde dentro de la VPC.

```bash
# 1) El balanceador, en las dos subredes de la default VPC
export ALB_ARN=$(aws elbv2 create-load-balancer \
  --name "$ALB_NAME" \
  --type application \
  --scheme internal \
  --subnets "$SUBNET_1" "$SUBNET_2" \
  --security-groups "$ALB_SG_ID" \
  --query "LoadBalancers[0].LoadBalancerArn" --output text)

# 2) Target group con destino "ip" (obligatorio para Fargate/awsvpc)
export TG_ARN=$(aws elbv2 create-target-group \
  --name "$TG_NAME" \
  --protocol HTTP --port 8000 \
  --vpc-id "$VPC_ID" \
  --target-type ip \
  --health-check-path /health \
  --query "TargetGroups[0].TargetGroupArn" --output text)

# 3) Listener en el 80 que reenvía al target group
export LISTENER_ARN=$(aws elbv2 create-listener \
  --load-balancer-arn "$ALB_ARN" \
  --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn="$TG_ARN" \
  --query "Listeners[0].ListenerArn" --output text)

echo "ALB:      $ALB_ARN"
echo "TG:       $TG_ARN"
echo "Listener: $LISTENER_ARN"
```

**Claves:**
- `--target-type ip`: en Fargate cada tarea tiene su propia ENI con IP privada; el target group registra IPs, no instancias EC2.
- `--health-check-path /health`: el ALB llamará a `GET /health` (que implementamos en el Bloque 1) para decidir si una tarea recibe tráfico.

---

## Fase 6 — ECS Fargate

### 6.1 Cluster y logs

```bash
aws ecs create-cluster --cluster-name "$CLUSTER_NAME"

aws logs create-log-group --log-group-name "/ecs/${PREFIX}-backend"
aws logs put-retention-policy \
  --log-group-name "/ecs/${PREFIX}-backend" --retention-in-days 7
```

El *log group* recogerá la salida de los contenedores (lo referenciaremos en la *task definition*). La retención de 7 días evita acumular coste de logs.

### 6.2 Roles IAM

ECS usa **dos roles** distintos, y entender la diferencia es uno de los objetivos de la práctica:

- **Rol de ejecución**: lo usa el *agente* de ECS para arrancar la tarea (descargar la imagen de ECR, leer el secreto, escribir logs).
- **Rol de tarea**: lo usa tu *aplicación* en ejecución (en el Bloque 3 le daremos permiso para publicar en SNS).

```bash
# Política de confianza común (ambos roles los asume el servicio ECS)
cat > ecs-trust.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ecs-tasks.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

# --- Rol de EJECUCIÓN ---
export EXEC_ROLE_ARN=$(aws iam create-role \
  --role-name "${PREFIX}-exec-role" \
  --assume-role-policy-document file://ecs-trust.json \
  --query "Role.Arn" --output text)

aws iam attach-role-policy \
  --role-name "${PREFIX}-exec-role" \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

# Permiso extra para leer el secreto de RDS
cat > exec-secret-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "secretsmanager:GetSecretValue",
    "Resource": "$DB_SECRET_ARN"
  }]
}
EOF
aws iam put-role-policy \
  --role-name "${PREFIX}-exec-role" \
  --policy-name read-db-secret \
  --policy-document file://exec-secret-policy.json

# --- Rol de TAREA (de momento sin permisos; en el Bloque 3 le añadimos SNS) ---
export TASK_ROLE_ARN=$(aws iam create-role \
  --role-name "${PREFIX}-task-role" \
  --assume-role-policy-document file://ecs-trust.json \
  --query "Role.Arn" --output text)

echo "Exec role: $EXEC_ROLE_ARN"
echo "Task role: $TASK_ROLE_ARN"
```

> El `AmazonECSTaskExecutionRolePolicy` cubre el *pull* desde ECR y la escritura de logs. La política `read-db-secret` permite **inyectar** las credenciales de RDS en el contenedor. El secreto está cifrado con la clave gestionada `aws/secretsmanager`, por lo que no hace falta permiso KMS adicional.

### 6.3 Task definition

Aquí está la parte interesante: las variables de BD no se escriben a mano. `DB_HOST`, `DB_NAME` y `AWS_REGION` van como `environment`, y **`DB_USER`/`DB_PASS` se leen del secreto** con la sintaxis `arn-del-secreto:clave-json::`.

```bash
cat > taskdef.json << EOF
{
  "family": "${TASK_FAMILY}",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "${EXEC_ROLE_ARN}",
  "taskRoleArn": "${TASK_ROLE_ARN}",
  "containerDefinitions": [
    {
      "name": "backend",
      "image": "${ECR_URI}:latest",
      "essential": true,
      "portMappings": [{"containerPort": 8000, "protocol": "tcp"}],
      "environment": [
        {"name": "DB_HOST", "value": "${DB_ENDPOINT}"},
        {"name": "DB_NAME", "value": "shopdb"},
        {"name": "DB_PORT", "value": "5432"},
        {"name": "AWS_REGION", "value": "${AWS_REGION}"}
      ],
      "secrets": [
        {"name": "DB_USER", "valueFrom": "${DB_SECRET_ARN}:username::"},
        {"name": "DB_PASS", "valueFrom": "${DB_SECRET_ARN}:password::"}
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/${PREFIX}-backend",
          "awslogs-region": "${AWS_REGION}",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
EOF

aws ecs register-task-definition --cli-input-json file://taskdef.json
```

> Si ves un error de tipo *role not found*, espera unos segundos (IAM tarda en propagar el rol recién creado) y vuelve a registrar la *task definition*.

### 6.4 Servicio ECS

El servicio mantiene la tarea en ejecución y la registra automáticamente en el target group del ALB.

```bash
aws ecs create-service \
  --cluster "$CLUSTER_NAME" \
  --service-name "$SERVICE_NAME" \
  --task-definition "$TASK_FAMILY" \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[$SUBNET_1,$SUBNET_2],securityGroups=[$ECS_SG_ID],assignPublicIp=ENABLED}" \
  --load-balancers "targetGroupArn=$TG_ARN,containerName=backend,containerPort=8000" \
  --health-check-grace-period-seconds 60
```

**Por qué `assignPublicIp=ENABLED`:** las subredes de la default VPC son públicas y no hay NAT Gateway. La tarea necesita IP pública para descargar la imagen de ECR. El *security group* sigue protegiéndola (solo acepta tráfico del ALB en el 8000).

Espera a que el servicio se estabilice y comprueba la salud de la tarea:

```bash
aws ecs wait services-stable --cluster "$CLUSTER_NAME" --services "$SERVICE_NAME"

aws elbv2 describe-target-health --target-group-arn "$TG_ARN" \
  --query "TargetHealthDescriptions[].TargetHealth.State" --output text
```

Debe devolver `healthy`. Si no, mira los logs del contenedor (suele ser un problema de conexión a la BD o de arranque):

```bash
aws logs tail "/ecs/${PREFIX}-backend" --since 10m
```

---

## Fase 7 — API Gateway (HTTP API + VPC Link)

El ALB es interno, así que para exponerlo creamos un **VPC Link**: un puente que permite a API Gateway alcanzar recursos privados dentro de la VPC.

### 7.1 VPC Link

```bash

# Evitamos un problema en una subred de us-east-1 no soporta

read -r SUBNET_1 SUBNET_2 <<< "$(aws ec2 describe-subnets \
  --filters Name=vpc-id,Values=$VPC_ID \
  --query "Subnets[?AvailabilityZoneId!='use1-az3'].SubnetId | [:2]" \
  --output text)"
export SUBNET_1 SUBNET_2
echo "Subredes para el VPC Link: $SUBNET_1 $SUBNET_2"


export VPC_LINK_ID=$(aws apigatewayv2 create-vpc-link \
  --name "${PREFIX}-vpclink" \
  --subnet-ids "$SUBNET_1" "$SUBNET_2" \
  --security-group-ids "$ALB_SG_ID" \
  --query "VpcLinkId" --output text)
echo "VPC_LINK_ID=$VPC_LINK_ID"


# El VPC Link tarda 1-2 minutos en quedar disponible
until [ "$(aws apigatewayv2 get-vpc-link --vpc-link-id "$VPC_LINK_ID" \
  --query VpcLinkStatus --output text)" = "AVAILABLE" ]; do
  echo "VPC Link creándose..."; sleep 15
done
echo "VPC Link disponible: $VPC_LINK_ID"
```

### 7.2 API, integración y ruta

```bash
# 1) La API HTTP
export API_ID=$(aws apigatewayv2 create-api \
  --name "${PREFIX}-api" \
  --protocol-type HTTP \
  --query "ApiId" --output text)

# 2) Integración privada: reenvía al listener del ALB a través del VPC Link
export INTEGRATION_ID=$(aws apigatewayv2 create-integration \
  --api-id "$API_ID" \
  --integration-type HTTP_PROXY \
  --integration-method ANY \
  --integration-uri "$LISTENER_ARN" \
  --connection-type VPC_LINK \
  --connection-id "$VPC_LINK_ID" \
  --payload-format-version 1.0 \
  --query "IntegrationId" --output text)

# 3) Ruta comodín: cualquier método, cualquier path -> la integración
aws apigatewayv2 create-route \
  --api-id "$API_ID" \
  --route-key 'ANY /{proxy+}' \
  --target "integrations/$INTEGRATION_ID"

# 4) Stage $default con auto-deploy (URL limpia, sin sufijo de stage)
aws apigatewayv2 create-stage \
  --api-id "$API_ID" \
  --stage-name '$default' \
  --auto-deploy
```

### 7.3 Probar de extremo a extremo

```bash
export API_ENDPOINT=$(aws apigatewayv2 get-api --api-id "$API_ID" \
  --query "ApiEndpoint" --output text)
echo "API: $API_ENDPOINT"

curl "$API_ENDPOINT/health"
curl "$API_ENDPOINT/products"
```

`/health` debe responder `{"status":"ok"}` y `/products` el catálogo de tecnología en JSON. Si lo ves, el camino completo funciona:

```
curl -> API Gateway -> VPC Link -> ALB interno -> ECS Fargate -> RDS PostgreSQL
```

---

## Ampliar setup-env.sh (ejecutar una sola vez)

Para que las próximas sesiones reconstruyan también los recursos del Bloque 2, añade su redescubrimiento al script:

```bash
cat >> setup-env.sh << 'EOF'

# ---- Bloque 2: cómputo y API (redescubrimiento) ----
export ALB_ARN=$(aws elbv2 describe-load-balancers --names "$ALB_NAME" \
  --query "LoadBalancers[0].LoadBalancerArn" --output text 2>/dev/null)
export TG_ARN=$(aws elbv2 describe-target-groups --names "$TG_NAME" \
  --query "TargetGroups[0].TargetGroupArn" --output text 2>/dev/null)
export LISTENER_ARN=$(aws elbv2 describe-listeners --load-balancer-arn "$ALB_ARN" \
  --query "Listeners[0].ListenerArn" --output text 2>/dev/null)
export EXEC_ROLE_ARN=$(aws iam get-role --role-name "${PREFIX}-exec-role" \
  --query "Role.Arn" --output text 2>/dev/null)
export TASK_ROLE_ARN=$(aws iam get-role --role-name "${PREFIX}-task-role" \
  --query "Role.Arn" --output text 2>/dev/null)
export API_ID=$(aws apigatewayv2 get-apis \
  --query "Items[?Name=='${PREFIX}-api'].ApiId | [0]" --output text 2>/dev/null)
export API_ENDPOINT=$(aws apigatewayv2 get-api --api-id "$API_ID" \
  --query "ApiEndpoint" --output text 2>/dev/null)
export VPC_LINK_ID=$(aws apigatewayv2 get-vpc-links \
  --query "Items[?Name=='${PREFIX}-vpclink'].VpcLinkId | [0]" --output text 2>/dev/null)
EOF
```

A partir de ahora, un simple `source setup-env.sh` deja también disponibles `API_ENDPOINT`, `TASK_ROLE_ARN`, `ALB_ARN`, etc.

---

## Checkpoint del Bloque 2

```bash
source setup-env.sh
echo "Servicio ECS:" && aws ecs describe-services --cluster "$CLUSTER_NAME" \
  --services "$SERVICE_NAME" --query "services[0].{estado:status,deseadas:desiredCount,activas:runningCount}"
echo "Salud target:" && aws elbv2 describe-target-health --target-group-arn "$TG_ARN" \
  --query "TargetHealthDescriptions[].TargetHealth.State" --output text
echo "API: $API_ENDPOINT"
curl -s "$API_ENDPOINT/products" | head -c 300; echo
```

- [x] ALB interno con target group `ip` y listener en el 80
- [x] Cluster ECS, log group y roles de ejecución/tarea
- [x] Task definition con credenciales de RDS inyectadas desde Secrets Manager
- [x] Servicio Fargate `healthy` y registrado en el ALB
- [x] API Gateway HTTP API publicando el backend vía VPC Link

**En el Bloque 3** crearemos el **topic SNS** y la **Lambda** de notificación, daremos al rol de tarea permiso para publicar, redesplegaremos el servicio con la nueva variable `SNS_TOPIC_ARN`, montaremos el **frontend HTML/JS en S3 + CloudFront** con imágenes de producto, haremos la prueba E2E completa (pedido → email) y dejaremos el **script de teardown** para borrarlo todo.
