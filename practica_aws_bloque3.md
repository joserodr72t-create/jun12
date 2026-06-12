# Práctica AWS — Tienda Master UCM
## Bloque 3: Asíncrono, autenticación y frontend (solución completa)

> **Punto de partida:** Bloques 1 y 2 completados (RDS, backend en ECS Fargate y API Gateway con VPC Link funcionando). 
>
> **En este bloque:** imágenes de producto en S3, **SNS** con doble suscripción (email + **Lambda**), permiso de publicación al backend y *redeploy*, **Cognito** (User Pool + cliente SDK) protegiendo el pedido, **frontend HTML/JS en S3 + CloudFront**, prueba de extremo a extremo y **teardown**.

Recarga variables al empezar:

```bash
source setup-env.sh
```

> Necesitas `jq` (incluido en CloudShell) para el frontend y el teardown.

---

## Ampliar setup-env.sh (ejecutar una sola vez, ANTES de empezar)

En este bloque los despliegues son largos y es fácil que la sesión de CloudShell expire a mitad. Para que un simple `source setup-env.sh` reconstruya también los recursos del Bloque 3, **amplía el script ahora**: las variables quedarán vacías hasta que crees cada recurso, y se rellenarán solas en cuanto existan.

```bash
cat >> setup-env.sh << 'EOF'

# ---- Bloque 3: async, auth y frontend (redescubrimiento) ----
export MEDIA_BASE_URL="https://${MEDIA_BUCKET}.s3.${AWS_REGION}.amazonaws.com"
export SNS_TOPIC_ARN=$(aws sns get-topic-attributes \
  --topic-arn "arn:aws:sns:${AWS_REGION}:${ACCOUNT_ID}:${SNS_TOPIC_NAME}" \
  --query "Attributes.TopicArn" --output text 2>/dev/null)
export LAMBDA_ARN=$(aws lambda get-function --function-name "$LAMBDA_NAME" \
  --query "Configuration.FunctionArn" --output text 2>/dev/null)
export USER_POOL_ID=$(aws cognito-idp list-user-pools --max-results 60 \
  --query "UserPools[?Name=='${PREFIX}-users'].Id | [0]" --output text 2>/dev/null)
export APP_CLIENT_ID=$(aws cognito-idp list-user-pool-clients --user-pool-id "$USER_POOL_ID" \
  --query "UserPoolClients[?ClientName=='${PREFIX}-web'].ClientId | [0]" --output text 2>/dev/null)
export OAC_ID=$(aws cloudfront list-origin-access-controls \
  --query "OriginAccessControlList.Items[?Name=='${PREFIX}-oac'].Id | [0]" --output text 2>/dev/null)
export DIST_ID=$(aws cloudfront list-distributions \
  --query "DistributionList.Items[?Comment=='Tienda Master UCM frontend'].Id | [0]" --output text 2>/dev/null)
EOF

source setup-env.sh
```


---

## Fase 8 — Imágenes de producto (bucket de media)

El catálogo del Bloque 1 referencia imágenes (`laptop.jpg`, `phone.jpg`, ...). Creamos un bucket de media de lectura pública y subimos *placeholders* etiquetados (libres de derechos, generados al vuelo). Puedes sustituirlos directamete por fotos reales (en imágenes.zip).

```bash
aws s3 mb "s3://$MEDIA_BUCKET"

# Permitir lectura pública de objetos en este bucket
aws s3api put-public-access-block --bucket "$MEDIA_BUCKET" \
  --public-access-block-configuration \
  BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false

cat > media-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "PublicReadMedia",
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::${MEDIA_BUCKET}/*"
  }]
}
EOF
aws s3api put-bucket-policy --bucket "$MEDIA_BUCKET" --policy file://media-policy.json

# OPCIONAL: Generar y subir un placeholder por producto (mismos nombres que el catálogo). Se pueden subir directamente las imágenes finales.
while IFS='|' read -r file label; do
  curl -s -o "$file" "https://placehold.co/600x400.jpg?text=${label}"
  aws s3 cp "$file" "s3://$MEDIA_BUCKET/$file" --content-type image/jpeg
done << 'LIST'
laptop.jpg|Portatil+UCM+Pro
phone.jpg|Smartphone+Campus+X
headphones.jpg|Auriculares+Aula+Quiet
monitor.jpg|Monitor+Master+27
keyboard.jpg|Teclado+Mecanico+DXC
tablet.jpg|Tablet+Beca+11
LIST

export MEDIA_BASE_URL="https://${MEDIA_BUCKET}.s3.${AWS_REGION}.amazonaws.com"
echo "Prueba una imagen: $MEDIA_BASE_URL/laptop.jpg"
```

---

## Fase 9 — SNS y Lambda

El backend publicará un evento al crear un pedido. El topic SNS hace **fan-out** a dos suscriptores: un **email** (notificación legible) y una **Lambda** (procesamiento programático). Es el patrón pub/sub clásico.

### 9.1 Topic y suscripción de email

```bash
export SNS_TOPIC_ARN=$(aws sns create-topic --name "$SNS_TOPIC_NAME" \
  --query "TopicArn" --output text)

# Pon TU correo aquí:
export ALUMNO_EMAIL="tu-correo@ejemplo.com"

aws sns subscribe --topic-arn "$SNS_TOPIC_ARN" \
  --protocol email --notification-endpoint "$ALUMNO_EMAIL"
```

> AWS te enviará un correo de confirmación. **Haz clic en "Confirm subscription"** o no recibirás las notificaciones. Revisa también la carpeta de **spam** (Gmail/Outlook lo filtran a menudo).

Comprueba el estado de la suscripción — si sigue en `PendingConfirmation`, los emails **no** llegarán aunque todo lo demás funcione:

```bash
aws sns list-subscriptions-by-topic --topic-arn "$SNS_TOPIC_ARN" \
  --query "Subscriptions[].{protocolo:Protocol,endpoint:Endpoint,estado:SubscriptionArn}" \
  --output table
```

Tras confirmar, la columna `estado` debe mostrar un ARN completo (no `PendingConfirmation`).

### 9.2 La función Lambda

```bash
mkdir -p lambda
cat > lambda/handler.py << 'EOF'
import json

def handler(event, context):
    for record in event.get("Records", []):
        mensaje = record["Sns"]["Message"]
        try:
            pedido = json.loads(mensaje)
        except Exception:
            pedido = {"raw": mensaje}
        print("Pedido procesado:", json.dumps(pedido, ensure_ascii=False))
        # En un sistema real: actualizar stock, llamar a pasarela de pago,
        # generar factura, etc.
    return {"status": "ok"}
EOF

cd lambda

zip -q function.zip handler.py 
 
cd ..
#volvemos al directorio superior desde donde lanzamos todos los comandos

# Rol de la Lambda (logs en CloudWatch)
cat > lambda-trust.json << 'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow",
"Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF
export LAMBDA_ROLE_ARN=$(aws iam create-role \
  --role-name "${PREFIX}-lambda-role" \
  --assume-role-policy-document file://lambda-trust.json \
  --query "Role.Arn" --output text)
aws iam attach-role-policy --role-name "${PREFIX}-lambda-role" \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Crear la función. IAM tarda unos segundos en propagar el rol recién creado:
# si create-function falla con "role cannot be assumed by Lambda", se reintenta.
sleep 10
while true; do
  LAMBDA_ARN=$(aws lambda create-function \
    --function-name "$LAMBDA_NAME" \
    --runtime python3.12 \
    --role "$LAMBDA_ROLE_ARN" \
    --handler handler.handler \
    --zip-file fileb://lambda/function.zip \
    --timeout 15 \
    --query "FunctionArn" --output text) && break
  echo "IAM aún propagando el rol; reintentando en 5s..."
  sleep 5
done
export LAMBDA_ARN
echo "LAMBDA_ARN=$LAMBDA_ARN"
```

### 9.3 Suscribir la Lambda al topic

```bash
# Permitir que SNS invoque la Lambda
aws lambda add-permission \
  --function-name "$LAMBDA_NAME" \
  --statement-id sns-invoke \
  --action lambda:InvokeFunction \
  --principal sns.amazonaws.com \
  --source-arn "$SNS_TOPIC_ARN"

# Suscripción tipo lambda
aws sns subscribe --topic-arn "$SNS_TOPIC_ARN" \
  --protocol lambda --notification-endpoint "$LAMBDA_ARN"
```

---

## Fase 10 — Conectar el backend a SNS (permiso + redeploy)

El backend ya trae el código para publicar en SNS (Bloque 1), pero está inactivo porque falta la variable `SNS_TOPIC_ARN` y el permiso. Le damos a la **tarea** permiso de publicación y registramos una **nueva revisión** de la *task definition* con dos variables nuevas (`SNS_TOPIC_ARN` y `MEDIA_BASE_URL`).

**Guard-rail antes de continuar.** Esta fase tiene una trampa: si llegas aquí con `SNS_TOPIC_ARN` vacía (por ejemplo, porque CloudShell expiró tras la Fase 9 y recargaste variables sin el redescubrimiento), la *task definition* **se registra sin ningún error** con la variable en blanco — a diferencia de un ARN de rol, una variable de entorno vacía es perfectamente válida para ECS. El backend arrancará, el pedido devolverá `"creado"`, y simplemente **nunca publicará en SNS**: ni email, ni Lambda, ni un solo error en ningún log. Es el peor tipo de fallo, el silencioso. Comprueba:

```bash
if [ -z "$SNS_TOPIC_ARN" ] || [ "$SNS_TOPIC_ARN" = "None" ]; then
  echo "⛔ SNS_TOPIC_ARN vacío. Ejecuta 'source setup-env.sh' (con el bloque 3 ya ampliado)"
  echo "   y verifica que el topic de la Fase 9 existe antes de seguir."
else
  echo "✅ SNS_TOPIC_ARN=$SNS_TOPIC_ARN"
  echo "✅ MEDIA_BASE_URL=$MEDIA_BASE_URL"
fi
```

Sigue solo si ves los dos ✅.

```bash
# 1) Permiso sns:Publish para el rol de tarea
cat > task-sns-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "sns:Publish",
    "Resource": "${SNS_TOPIC_ARN}"
  }]
}
EOF
aws iam put-role-policy --role-name "${PREFIX}-task-role" \
  --policy-name publish-orders --policy-document file://task-sns-policy.json

# 2) Nueva revisión de la task definition (añade SNS_TOPIC_ARN y MEDIA_BASE_URL)
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
        {"name": "AWS_REGION", "value": "${AWS_REGION}"},
        {"name": "SNS_TOPIC_ARN", "value": "${SNS_TOPIC_ARN}"},
        {"name": "MEDIA_BASE_URL", "value": "${MEDIA_BASE_URL}"}
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

# 3) Forzar el redeploy del servicio con la nueva revisión
aws ecs update-service --cluster "$CLUSTER_NAME" --service "$SERVICE_NAME" \
  --task-definition "$TASK_FAMILY" --force-new-deployment
aws ecs wait services-stable --cluster "$CLUSTER_NAME" --services "$SERVICE_NAME"
```

ECS hace un *rolling deployment*: levanta la tarea nueva, espera a que esté sana en el ALB y retira la antigua, sin cortar el servicio. Las imágenes del catálogo ahora se sirven con URL completa (`MEDIA_BASE_URL`).

Verifica que la revisión **desplegada** lleva el ARN del topic (es la comprobación definitiva de esta fase):

```bash
aws ecs describe-task-definition \
  --task-definition "$(aws ecs describe-services --cluster "$CLUSTER_NAME" --services "$SERVICE_NAME" \
      --query "services[0].taskDefinition" --output text)" \
  --query "taskDefinition.{revision:revision, sns:containerDefinitions[0].environment[?name=='SNS_TOPIC_ARN'].value | [0]}"
```

El campo `sns` debe mostrar el ARN completo. Si sale vacío o la revisión es la del Bloque 2, el redeploy usó una revisión sin la variable: vuelve al *guard-rail* del inicio de esta fase.

---

## Fase 11 — Cognito (User Pool + cliente para SDK)

```bash
# User pool: login por email, verificación automática, política de contraseña
export USER_POOL_ID=$(aws cognito-idp create-user-pool \
  --pool-name "${PREFIX}-users" \
  --auto-verified-attributes email \
  --username-attributes email \
  --policies '{"PasswordPolicy":{"MinimumLength":8,"RequireUppercase":true,"RequireLowercase":true,"RequireNumbers":true,"RequireSymbols":false}}' \
  --query "UserPool.Id" --output text)

# App client SIN secreto (obligatorio para JS de navegador) con flujos SRP y password
export APP_CLIENT_ID=$(aws cognito-idp create-user-pool-client \
  --user-pool-id "$USER_POOL_ID" \
  --client-name "${PREFIX}-web" \
  --no-generate-secret \
  --explicit-auth-flows ALLOW_USER_SRP_AUTH ALLOW_USER_PASSWORD_AUTH ALLOW_REFRESH_TOKEN_AUTH \
  --query "UserPoolClient.ClientId" --output text)

echo "USER_POOL_ID=$USER_POOL_ID"
echo "APP_CLIENT_ID=$APP_CLIENT_ID"

# Usuario de prueba ya confirmado (para demostrar sin esperar al email de verificación)
aws cognito-idp admin-create-user --user-pool-id "$USER_POOL_ID" \
  --username "alumno@ejemplo.com" --message-action SUPPRESS \
  --user-attributes Name=email,Value=alumno@ejemplo.com Name=email_verified,Value=true
aws cognito-idp admin-set-user-password --user-pool-id "$USER_POOL_ID" \
  --username "alumno@ejemplo.com" --password 'TiendaUCM2026' --permanent
```

**Por qué `--no-generate-secret`:** las apps de navegador no pueden guardar un secreto de cliente; el SDK usa el protocolo SRP (no envía la contraseña en claro). El usuario de prueba evita el ida y vuelta del email de verificación durante la demo, pero el registro normal (con código por email) también funciona.

---

## Fase 12 — Proteger el pedido con un authorizer JWT

Añadimos un *authorizer* JWT y una ruta **específica** `POST /orders` asociada a él. Como API Gateway elige siempre la ruta más concreta, `POST /orders` pasará por Cognito mientras el resto sigue público.

```bash
# Authorizer JWT apuntando al User Pool
export AUTHORIZER_ID=$(aws apigatewayv2 create-authorizer \
  --api-id "$API_ID" \
  --authorizer-type JWT \
  --name "${PREFIX}-cognito" \
  --identity-source '$request.header.Authorization' \
  --jwt-configuration "Audience=${APP_CLIENT_ID},Issuer=https://cognito-idp.${AWS_REGION}.amazonaws.com/${USER_POOL_ID}" \
  --query "AuthorizerId" --output text)

# Reutilizar la integración del VPC Link ya creada en el Bloque 2
export INTEGRATION_ID=$(aws apigatewayv2 get-integrations --api-id "$API_ID" \
  --query "Items[0].IntegrationId" --output text)

# Ruta protegida específica para crear pedidos
aws apigatewayv2 create-route \
  --api-id "$API_ID" \
  --route-key 'POST /orders' \
  --target "integrations/${INTEGRATION_ID}" \
  --authorization-type JWT \
  --authorizer-id "$AUTHORIZER_ID"
```

> El *authorizer* valida la **audiencia** (`aud`) del token, que en Cognito coincide con el **ID token** (no el *access token*). Por eso el frontend enviará el ID token. El *preflight* `OPTIONS /orders` que manda el navegador cae en la ruta comodín (pública), así que CORS no se rompe.

---

## Fase 13 — Frontend (HTML/JS + Cognito SDK) en S3 + CloudFront

### 13.1 Configuración dinámica

Generamos un `config.js` con los valores reales (este sí expande variables):

```bash

mkdir -p frontend 
cd frontend

cat > config.js << EOF
window.SHOP_CONFIG = {
  region: "${AWS_REGION}",
  userPoolId: "${USER_POOL_ID}",
  clientId: "${APP_CLIENT_ID}",
  apiEndpoint: "${API_ENDPOINT}"
};
EOF
```

### 13.2 La página

Catálogo público; el botón de pedido se habilita solo tras iniciar sesión. Se escribe con *here-doc entrecomillado* (`'HTML'`) para que el JavaScript no lo toque la shell:

```bash

cat > index.html << 'HTML'
<!DOCTYPE html>
<html lang="es">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Tienda Master UCM</title>
<style>
  :root{--azul:#0c447c;}
  *{box-sizing:border-box;font-family:Arial,Helvetica,sans-serif;}
  body{margin:0;background:#fafafa;color:#222;}
  header{background:var(--azul);color:#fff;padding:16px 24px;display:flex;
    justify-content:space-between;align-items:center;}
  header h1{font-size:20px;margin:0;}
  main{max-width:1000px;margin:24px auto;padding:0 16px;}
  .auth,.cart{background:#fff;border:1px solid #ddd;border-radius:8px;padding:16px;}
  .auth{margin-bottom:24px;}
  input{padding:8px;margin-right:8px;border:1px solid #ccc;border-radius:4px;}
  button{background:var(--azul);color:#fff;border:0;border-radius:4px;
    padding:8px 14px;cursor:pointer;}
  button:disabled{opacity:.5;cursor:not-allowed;}
  .grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(220px,1fr));gap:16px;}
  .card{background:#fff;border:1px solid #ddd;border-radius:8px;overflow:hidden;}
  .card img{width:100%;height:150px;object-fit:cover;}
  .card .body{padding:12px;}
  .card h3{font-size:15px;margin:0 0 4px;}
  .price{font-weight:bold;color:var(--azul);}
  .cart{position:sticky;top:16px;margin-top:24px;}
  .msg{padding:10px;border-radius:4px;margin-top:8px;}
  .ok{background:#e1f5ee;color:#0f6e56;}
  .err{background:#fcebeb;color:#a32d2d;}
  small{color:#666;}
</style>
</head>
<body>
<header>
  <h1>Tienda Master UCM</h1>
  <span id="userbar"><small>No has iniciado sesion</small></span>
</header>
<main>
  <section class="auth">
    <strong>Acceso</strong> <small>(necesario para comprar)</small><br><br>
    <input id="email" type="email" placeholder="email">
    <input id="password" type="password" placeholder="contraseña">
    <button id="loginBtn">Entrar</button>
    <button id="logoutBtn" style="display:none;background:#888">Salir</button>
    <div id="authMsg"></div>
  </section>

  <h2>Catálogo</h2>
  <div class="grid" id="grid">Cargando...</div>

  <div class="cart">
    <strong>Carrito</strong>
    <div id="cartList"><small>Vacio</small></div>
    <p>Total: <span id="total">0.00</span> &euro;</p>
    <button id="orderBtn" disabled>Realizar pedido</button>
    <div id="orderMsg"></div>
  </div>
</main>

<script src="https://cdn.jsdelivr.net/npm/amazon-cognito-identity-js@6/dist/amazon-cognito-identity.min.js"></script>
<script src="config.js"></script>
<script>
const { CognitoUserPool, CognitoUser, AuthenticationDetails } = AmazonCognitoIdentity;
const cfg = window.SHOP_CONFIG;
const pool = new CognitoUserPool({ UserPoolId: cfg.userPoolId, ClientId: cfg.clientId });
const $ = (id) => document.getElementById(id);
let idToken = null, email = null, products = {}, cart = {};

function setAuthUI(){
  $("logoutBtn").style.display = idToken ? "" : "none";
  $("loginBtn").style.display  = idToken ? "none" : "";
  $("userbar").innerHTML = idToken
    ? "<small>Sesion: " + email + "</small>"
    : "<small>No has iniciado sesion</small>";
  updateCart();
}

$("loginBtn").onclick = function(){
  const e = $("email").value, p = $("password").value;
  const user = new CognitoUser({ Username: e, Pool: pool });
  const auth = new AuthenticationDetails({ Username: e, Password: p });
  user.authenticateUser(auth, {
    onSuccess: function(res){
      idToken = res.getIdToken().getJwtToken(); email = e;
      $("authMsg").innerHTML = '<div class="msg ok">Sesion iniciada</div>';
      setAuthUI();
    },
    onFailure: function(err){
      $("authMsg").innerHTML = '<div class="msg err">' + err.message + '</div>';
    }
  });
};

$("logoutBtn").onclick = function(){
  idToken = null; email = null; $("authMsg").innerHTML = ""; setAuthUI();
};

function loadProducts(){
  fetch(cfg.apiEndpoint + "/products")
    .then(function(r){ return r.json(); })
    .then(function(items){
      products = {};
      items.forEach(function(p){ products[p.id] = p; });
      $("grid").innerHTML = items.map(function(p){
        return '<div class="card"><img src="' + p.image + '" alt="">' +
          '<div class="body"><h3>' + p.name + '</h3>' +
          '<small>' + p.description + '</small>' +
          '<p class="price">' + p.price.toFixed(2) + ' &euro;</p>' +
          '<button data-add="' + p.id + '">Añadir</button></div></div>';
      }).join("");
      Array.prototype.forEach.call($("grid").querySelectorAll("[data-add]"),
        function(b){ b.onclick = function(){ addToCart(Number(b.getAttribute("data-add"))); }; });
    })
    .catch(function(){ $("grid").innerHTML =
      '<div class="msg err">No se pudo cargar el catálogo</div>'; });
}

function addToCart(id){
  const p = products[id];
  if(!cart[id]) cart[id] = { product_id:id, name:p.name, price:p.price, qty:0 };
  cart[id].qty++;
  updateCart();
}

function updateCart(){
  const ids = Object.keys(cart);
  let total = 0;
  $("cartList").innerHTML = ids.length ? ids.map(function(id){
    const it = cart[id]; total += it.price * it.qty;
    return "<div>" + it.qty + "x " + it.name + " - " +
      (it.price * it.qty).toFixed(2) + " &euro;</div>";
  }).join("") : "<small>Vacio</small>";
  $("total").textContent = total.toFixed(2);
  $("orderBtn").disabled = !(idToken && ids.length);
}

$("orderBtn").onclick = function(){
  const items = Object.keys(cart).map(function(id){
    return { product_id: cart[id].product_id, qty: cart[id].qty };
  });
  fetch(cfg.apiEndpoint + "/orders", {
    method: "POST",
    headers: { "Content-Type": "application/json", "Authorization": "Bearer " + idToken },
    body: JSON.stringify({ email: email, items: items })
  })
  .then(function(r){ return r.ok ? r.json() : Promise.reject(r.status); })
  .then(function(res){
    $("orderMsg").innerHTML = '<div class="msg ok">Pedido ' + res.order_id +
      ' creado. Total ' + res.total.toFixed(2) + ' &euro;</div>';
    cart = {}; updateCart();
  })
  .catch(function(code){
    $("orderMsg").innerHTML = '<div class="msg err">Error al crear el pedido (' + code + ')</div>';
  });
};

loadProducts();
setAuthUI();
</script>
</body>
</html>
HTML
```

### 13.3 Bucket privado + CloudFront con OAC

El bucket del frontend queda **privado**; solo CloudFront puede leerlo, mediante *Origin Access Control* (OAC). CloudFront además da HTTPS.

```bash
aws s3 mb "s3://$FRONTEND_BUCKET"
aws s3 cp index.html "s3://$FRONTEND_BUCKET/index.html" --content-type text/html
aws s3 cp config.js  "s3://$FRONTEND_BUCKET/config.js"  --content-type application/javascript

# 1) Origin Access Control
cat > oac.json << EOF
{"Name":"${PREFIX}-oac","Description":"OAC Tienda UCM",
 "SigningProtocol":"sigv4","SigningBehavior":"always","OriginAccessControlOriginType":"s3"}
EOF
export OAC_ID=$(aws cloudfront create-origin-access-control \
  --origin-access-control-config file://oac.json \
  --query "OriginAccessControl.Id" --output text)

# 2) Distribución (CachingOptimized = 658327ea-... es una policy gestionada)
cat > cf-dist.json << EOF
{
  "CallerReference": "${PREFIX}-$(date +%s)",
  "Comment": "Tienda Master UCM frontend",
  "Enabled": true,
  "DefaultRootObject": "index.html",
  "Origins": { "Quantity": 1, "Items": [{
    "Id": "s3-frontend",
    "DomainName": "${FRONTEND_BUCKET}.s3.${AWS_REGION}.amazonaws.com",
    "OriginAccessControlId": "${OAC_ID}",
    "S3OriginConfig": { "OriginAccessIdentity": "" }
  }]},
  "DefaultCacheBehavior": {
    "TargetOriginId": "s3-frontend",
    "ViewerProtocolPolicy": "redirect-to-https",
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
    "AllowedMethods": { "Quantity": 2, "Items": ["GET","HEAD"] }
  }
}
EOF
export DIST_ID=$(aws cloudfront create-distribution \
  --distribution-config file://cf-dist.json --query "Distribution.Id" --output text)
export DIST_DOMAIN=$(aws cloudfront get-distribution --id "$DIST_ID" \
  --query "Distribution.DomainName" --output text)

# 3) Permitir a ESTA distribución leer el bucket
cat > frontend-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCloudFront",
    "Effect": "Allow",
    "Principal": { "Service": "cloudfront.amazonaws.com" },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::${FRONTEND_BUCKET}/*",
    "Condition": { "StringEquals": {
      "AWS:SourceArn": "arn:aws:cloudfront::${ACCOUNT_ID}:distribution/${DIST_ID}" } }
  }]
}
EOF
aws s3api put-bucket-policy --bucket "$FRONTEND_BUCKET" --policy file://frontend-policy.json

echo "Esperando a que CloudFront despliegue (puede tardar 5-10 min)..."
aws cloudfront wait distribution-deployed --id "$DIST_ID"
echo "TIENDA: https://$DIST_DOMAIN"
```

---

## Fase 14 — Prueba de extremo a extremo

### 14.1 El pedido exige autenticación (401 sin token)

```bash
curl -s -o /dev/null -w "HTTP %{http_code}\n" -X POST "$API_ENDPOINT/orders" \
  -H "Content-Type: application/json" \
  -d '{"email":"x@x.com","items":[{"product_id":1,"qty":1}]}'
# -> HTTP 401  (API Gateway lo rechaza antes de llegar al backend)
```

### 14.2 Con token sí funciona

Conseguimos un ID token del usuario de prueba (flujo `USER_PASSWORD_AUTH`) y repetimos:

```bash
export ID_TOKEN=$(aws cognito-idp initiate-auth \
  --auth-flow USER_PASSWORD_AUTH \
  --client-id "$APP_CLIENT_ID" \
  --auth-parameters USERNAME=alumno@ejemplo.com,PASSWORD='TiendaUCM2026' \
  --query "AuthenticationResult.IdToken" --output text)

curl -s -X POST "$API_ENDPOINT/orders" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $ID_TOKEN" \
  -d '{"email":"alumno@ejemplo.com","items":[{"product_id":1,"qty":1},{"product_id":3,"qty":2}]}'
# -> {"order_id":1,"total":1497.0,"status":"creado"}
```

### 14.3 Comprobar que SNS y Lambda reaccionaron

```bash
# La Lambda debe haber registrado el pedido
aws logs tail "/aws/lambda/${LAMBDA_NAME}" --since 5m
```

Y deberías recibir el **email** de notificación (si confirmaste la suscripción).

**Si el pedido se crea pero no llega ni el email ni el log de la Lambda**, diagnostica en este orden:

```bash
# 1) ¿La revisión activa lleva SNS_TOPIC_ARN? (la trampa de la Fase 10)
aws ecs describe-task-definition \
  --task-definition "$(aws ecs describe-services --cluster "$CLUSTER_NAME" --services "$SERVICE_NAME" \
      --query "services[0].taskDefinition" --output text)" \
  --query "taskDefinition.containerDefinitions[0].environment[?name=='SNS_TOPIC_ARN'].value"

# 2) ¿La suscripción de email está confirmada? (debe ser un ARN, no PendingConfirmation)
aws sns list-subscriptions-by-topic --topic-arn "$SNS_TOPIC_ARN" \
  --query "Subscriptions[].{protocolo:Protocol,estado:SubscriptionArn}" --output table

# 3) ¿El backend lanzó un error al publicar? (p. ej. AccessDenied si falta sns:Publish)
aws logs tail "/ecs/${PREFIX}-backend" --since 10m
```

Tres causas, tres síntomas distintos: (1) variable vacía = fallo **silencioso**, todo parece funcionar; (2) suscripción pendiente = la Lambda sí se ejecuta pero el email no llega; (3) permiso ausente = el pedido se guarda pero el cliente recibe un **500** (la excepción de boto3 salta después del `commit`).

### 14.4 La tienda completa

Abre `https://$DIST_DOMAIN` en el navegador: verás el catálogo con imágenes; inicia sesión con `alumno@ejemplo.com` / `TiendaUCM2026`, añade productos y pulsa **Realizar pedido**. El flujo completo:

```
Navegador (CloudFront) --login--> Cognito
       |
       | catálogo (publico)            pedido + Bearer token
       v                                      v
   API Gateway --/products--> ALB -> ECS   API Gateway --(JWT OK)--> ALB -> ECS -> RDS
                                                                            |
                                                                            v
                                                                    SNS --> email
                                                                        --> Lambda
```

---

## Fase 15 — Teardown (borrar todo)

> ⚠️ Borra **todos** los recursos para no acumular coste. CloudFront es el más lento (hay que deshabilitarlo antes de borrarlo). Requiere `jq`.

```bash
source setup-env.sh

# 1) CloudFront: deshabilitar, esperar y borrar
ETAG=$(aws cloudfront get-distribution-config --id "$DIST_ID" --query ETag --output text)
aws cloudfront get-distribution-config --id "$DIST_ID" --query DistributionConfig \
  | jq '.Enabled=false' > cf-disabled.json
aws cloudfront update-distribution --id "$DIST_ID" \
  --distribution-config file://cf-disabled.json --if-match "$ETAG"
aws cloudfront wait distribution-deployed --id "$DIST_ID"
ETAG=$(aws cloudfront get-distribution-config --id "$DIST_ID" --query ETag --output text)
aws cloudfront delete-distribution --id "$DIST_ID" --if-match "$ETAG"
aws cloudfront delete-origin-access-control --id "$OAC_ID" \
  --if-match "$(aws cloudfront get-origin-access-control --id "$OAC_ID" --query ETag --output text)" || true

# 2) S3
aws s3 rb "s3://$FRONTEND_BUCKET" --force || true
aws s3 rb "s3://$MEDIA_BUCKET" --force || true

# 3) API Gateway (borrar la API elimina rutas, integraciones y authorizers) + VPC Link
aws apigatewayv2 delete-api --api-id "$API_ID" || true
sleep 5
aws apigatewayv2 delete-vpc-link --vpc-link-id "$VPC_LINK_ID" || true

# 4) ECS
aws ecs update-service --cluster "$CLUSTER_NAME" --service "$SERVICE_NAME" --desired-count 0 || true
aws ecs delete-service --cluster "$CLUSTER_NAME" --service "$SERVICE_NAME" --force || true
aws ecs delete-cluster --cluster "$CLUSTER_NAME" || true

# 5) ALB + target group
aws elbv2 delete-load-balancer --load-balancer-arn "$ALB_ARN" || true
sleep 30
aws elbv2 delete-target-group --target-group-arn "$TG_ARN" || true

# 6) RDS (sin snapshot final) + subnet group + borra su secreto gestionado
aws rds delete-db-instance --db-instance-identifier "$DB_IDENTIFIER" \
  --skip-final-snapshot --delete-automated-backups || true
aws rds wait db-instance-deleted --db-instance-identifier "$DB_IDENTIFIER" || true
aws rds delete-db-subnet-group --db-subnet-group-name "$DB_SUBNET_GROUP" || true

# 7) SNS, Lambda y logs
aws sns delete-topic --topic-arn "$SNS_TOPIC_ARN" || true
aws lambda delete-function --function-name "$LAMBDA_NAME" || true
aws logs delete-log-group --log-group-name "/ecs/${PREFIX}-backend" || true
aws logs delete-log-group --log-group-name "/aws/lambda/${LAMBDA_NAME}" || true

# 8) Cognito
aws cognito-idp delete-user-pool --user-pool-id "$USER_POOL_ID" || true

# 9) ECR
aws ecr delete-repository --repository-name "$ECR_REPO" --force || true

# 10) Roles IAM
aws iam detach-role-policy --role-name "${PREFIX}-exec-role" \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy || true
aws iam delete-role-policy --role-name "${PREFIX}-exec-role" --policy-name read-db-secret || true
aws iam delete-role --role-name "${PREFIX}-exec-role" || true
aws iam delete-role-policy --role-name "${PREFIX}-task-role" --policy-name publish-orders || true
aws iam delete-role --role-name "${PREFIX}-task-role" || true
aws iam detach-role-policy --role-name "${PREFIX}-lambda-role" \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole || true
aws iam delete-role --role-name "${PREFIX}-lambda-role" || true

# 11) Security groups (en orden: RDS -> ECS -> ALB)
aws ec2 delete-security-group --group-id "$RDS_SG_ID" || true
aws ec2 delete-security-group --group-id "$ECS_SG_ID" || true
aws ec2 delete-security-group --group-id "$ALB_SG_ID" || true

echo "Teardown completado. Revisa la consola por si algún recurso tardó en borrarse."
```

> Si algún *security group* falla al borrarse, suele ser porque el recurso que lo usaba (ALB/RDS) aún se está eliminando: espera un minuto y reejecuta esas tres líneas.

---

## Resumen de la práctica completa

Has construido, solo con AWS CLI, una tienda de tecnología con arquitectura desacoplada y de producción:

- **Bloque 1:** variables deterministas, default VPC, *security groups* encadenados, RDS PostgreSQL con secreto gestionado, imagen del backend en ECR.
- **Bloque 2:** ALB interno, ECS Fargate con credenciales inyectadas desde Secrets Manager, API Gateway con VPC Link.
- **Bloque 3:** imágenes en S3, SNS con fan-out a email y Lambda, Cognito protegiendo el pedido vía JWT, frontend en S3 + CloudFront, y teardown completo.

Conceptos clave demostrados: separación de responsabilidades, *least privilege* en IAM y *security groups*, gestión de secretos, mensajería desacoplada (pub/sub), autenticación con tokens JWT y entrega de contenido estático con CDN.
