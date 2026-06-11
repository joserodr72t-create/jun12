# Práctica AWS — Tienda Master UCM
## Bloque 1: Fundamentos (variables, red, datos y backend)

> **Qué vamos a construir (los tres bloques)**
> Una tienda de tecnología desplegada paso a paso por **AWS CLI**, con arquitectura desacoplada:
> frontend estático en **S3 + CloudFront** → **API Gateway** → **ALB interno** → backend en **ECS Fargate** → **RDS PostgreSQL**, con imágenes en **S3** y notificaciones de pedido vía **SNS → Lambda**.
>
> **Este bloque (1)** deja lista la base: variables de entorno, descubrimiento de la *default VPC*, los *security groups*, la base de datos **RDS PostgreSQL** y la imagen del backend publicada en **ECR**.

---

## 0. Prerrequisitos

- **AWS CLI v2** configurada (`aws configure`) con un usuario/rol con permisos de administrador sobre la cuenta de laboratorio.
- **Docker** para la Fase 4 (construir la imagen del backend). En local: Docker Desktop. En **AWS CloudShell** comprueba con `docker --version`; si no está, se instala con:
  ```bash
  sudo dnf install -y docker && sudo systemctl enable --now docker
  ```
- Opcional: `jq` para inspeccionar salidas JSON.

Comprueba que la CLI te reconoce:

```bash
aws sts get-caller-identity
```

Debe devolver tu `Account`, `Arn` y `UserId`. Si falla, revisa `aws configure` antes de seguir.

---

## Fase 0 — Variables de entorno

Toda la práctica usa variables para que **nadie tenga que inventar nombres** y para que los recursos no choquen entre alumnos. La clave es que los nombres son **deterministas** (se derivan del *account id* y de la fecha) y los IDs de recursos ya creados se **redescubren** consultando a AWS. Así, cada vez que abras una terminal nueva, basta con volver a cargar el script.

Crea el archivo `setup-env.sh`:

```bash
cat > setup-env.sh << 'EOF'
#!/usr/bin/env bash
# ====== Tienda Master UCM — variables de entorno ======
# Re-ejecutable: redescubre los IDs de recursos ya creados.
# Uso:   source setup-env.sh

# 1) Región: cada alumno elige la suya (por defecto eu-west-1)
export AWS_REGION="${AWS_REGION:-eu-west-1}"
export AWS_DEFAULT_REGION="$AWS_REGION"

# 2) Identidad de la cuenta (vía STS)
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# 3) Fecha fija del despliegue (se guarda la primera vez en .shop-fecha)
if [ -f .shop-fecha ]; then
  export FECHA=$(cat .shop-fecha)
else
  export FECHA=$(date +%Y%m%d)
  echo "$FECHA" > .shop-fecha
fi

# 4) Naming determinista
export PREFIX="ucm-shop"
export FRONTEND_BUCKET="${PREFIX}-frontend-${FECHA}-${ACCOUNT_ID}"
export MEDIA_BUCKET="${PREFIX}-media-${FECHA}-${ACCOUNT_ID}"
export ECR_REPO="${PREFIX}-backend"
export DB_IDENTIFIER="${PREFIX}-db"
export DB_SUBNET_GROUP="${PREFIX}-db-subnets"
export CLUSTER_NAME="${PREFIX}-cluster"
export SERVICE_NAME="${PREFIX}-svc"
export TASK_FAMILY="${PREFIX}-task"
export ALB_NAME="${PREFIX}-alb"
export TG_NAME="${PREFIX}-tg"
export SNS_TOPIC_NAME="${PREFIX}-pedidos"
export LAMBDA_NAME="${PREFIX}-notificador"
export ECR_URI="${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"

# 5) Red: VPC por defecto y sus dos primeras subredes
export VPC_ID=$(aws ec2 describe-vpcs \
  --filters Name=isDefault,Values=true \
  --query "Vpcs[0].VpcId" --output text)
export VPC_CIDR=$(aws ec2 describe-vpcs --vpc-ids "$VPC_ID" \
  --query "Vpcs[0].CidrBlock" --output text)
read -r SUBNET_1 SUBNET_2 <<< "$(aws ec2 describe-subnets \
  --filters Name=vpc-id,Values=$VPC_ID \
  --query 'Subnets[:2].SubnetId' --output text)"
export SUBNET_1 SUBNET_2

# 6) IDs de recursos ya existentes (quedan vacíos si aún no se han creado)
export ALB_SG_ID=$(aws ec2 describe-security-groups \
  --filters Name=group-name,Values=${PREFIX}-alb-sg Name=vpc-id,Values=$VPC_ID \
  --query "SecurityGroups[0].GroupId" --output text 2>/dev/null)
export ECS_SG_ID=$(aws ec2 describe-security-groups \
  --filters Name=group-name,Values=${PREFIX}-ecs-sg Name=vpc-id,Values=$VPC_ID \
  --query "SecurityGroups[0].GroupId" --output text 2>/dev/null)
export RDS_SG_ID=$(aws ec2 describe-security-groups \
  --filters Name=group-name,Values=${PREFIX}-rds-sg Name=vpc-id,Values=$VPC_ID \
  --query "SecurityGroups[0].GroupId" --output text 2>/dev/null)
export DB_ENDPOINT=$(aws rds describe-db-instances \
  --db-instance-identifier "$DB_IDENTIFIER" \
  --query "DBInstances[0].Endpoint.Address" --output text 2>/dev/null)
export DB_SECRET_ARN=$(aws rds describe-db-instances \
  --db-instance-identifier "$DB_IDENTIFIER" \
  --query "DBInstances[0].MasterUserSecret.SecretArn" --output text 2>/dev/null)

echo "Región:      $AWS_REGION"
echo "Cuenta:      $ACCOUNT_ID"
echo "VPC:         $VPC_ID ($VPC_CIDR)"
echo "Subredes:    $SUBNET_1  $SUBNET_2"
echo "Bucket front:$FRONTEND_BUCKET"
EOF
```

Cárgalo (y repítelo al empezar **cada bloque** o al abrir una terminal nueva):

```bash
source setup-env.sh
```

**Qué hace y por qué:**
- `aws sts get-caller-identity` resuelve el *account id* sin que el alumno lo teclee.
- `.shop-fecha` **congela** la fecha del primer despliegue, de modo que los nombres de bucket (que son globales y únicos) no cambien si vuelves otro día.
- El bloque 5 lee la **default VPC** y sus subredes en vez de crearlas: nos ahorramos VPC propia, subredes y NAT Gateway (menos pasos y menos coste).
- El bloque 6 **redescubre** IDs (`ALB_SG_ID`, `DB_ENDPOINT`, etc.). Al ser re-ejecutable, nunca pierdes el estado entre sesiones: lo reconstruyes consultando a AWS.

---

## Fase 1 — Verificar la red (default VPC)

No creamos red: usamos la que ya trae la cuenta. Solo comprobamos que tenemos VPC y al menos dos subredes en zonas distintas (las necesita el ALB y el *subnet group* de RDS).

```bash
echo "VPC:      $VPC_ID"
echo "CIDR:     $VPC_CIDR"
echo "Subred 1: $SUBNET_1"
echo "Subred 2: $SUBNET_2"
```

Las subredes de la *default VPC* son públicas (tienen ruta al Internet Gateway). Eso nos permite que las tareas Fargate descarguen la imagen desde ECR con IP pública, **sin NAT**. La base de datos, en cambio, la dejaremos **sin acceso público** y protegida por *security group*.

---

## Fase 2 — Security groups (capas de seguridad)

Creamos tres grupos de seguridad y los **encadenamos**, de forma que cada capa solo acepta tráfico de la capa anterior:

```
API Gateway (VPC Link) --:80--> [ALB SG] --:8000--> [ECS SG] --:5432--> [RDS SG]
```

Crea los tres grupos:

```bash
export ALB_SG_ID=$(aws ec2 create-security-group \
  --group-name "${PREFIX}-alb-sg" \
  --description "ALB interno - Tienda Master UCM" \
  --vpc-id "$VPC_ID" --query GroupId --output text)

export ECS_SG_ID=$(aws ec2 create-security-group \
  --group-name "${PREFIX}-ecs-sg" \
  --description "Tareas ECS Fargate - Tienda Master UCM" \
  --vpc-id "$VPC_ID" --query GroupId --output text)

export RDS_SG_ID=$(aws ec2 create-security-group \
  --group-name "${PREFIX}-rds-sg" \
  --description "RDS PostgreSQL - Tienda Master UCM" \
  --vpc-id "$VPC_ID" --query GroupId --output text)

echo "ALB_SG_ID=$ALB_SG_ID  ECS_SG_ID=$ECS_SG_ID  RDS_SG_ID=$RDS_SG_ID"
```

Ahora las reglas de entrada (*ingress*):

```bash
# El ALB recibe tráfico HTTP solo desde dentro de la VPC (lo enviará el VPC Link de API Gateway)
aws ec2 authorize-security-group-ingress \
  --group-id "$ALB_SG_ID" \
  --protocol tcp --port 80 --cidr "$VPC_CIDR"

# El contenedor (puerto 8000) solo acepta tráfico que venga del ALB
aws ec2 authorize-security-group-ingress \
  --group-id "$ECS_SG_ID" \
  --protocol tcp --port 8000 --source-group "$ALB_SG_ID"

# La base de datos (5432) solo acepta tráfico que venga de las tareas ECS
aws ec2 authorize-security-group-ingress \
  --group-id "$RDS_SG_ID" \
  --protocol tcp --port 5432 --source-group "$ECS_SG_ID"
```

**Por qué así:** usar `--source-group` (en vez de un CIDR amplio) implementa el *least privilege*: aunque alguien conociera el endpoint de la base de datos, no podría conectarse si su tráfico no nace dentro del *security group* de ECS. Es el patrón de referencia para microservicios en AWS.

---

## Fase 3 — Base de datos RDS PostgreSQL

Primero, un **DB subnet group** con las dos subredes (RDS exige subredes en al menos dos zonas):

```bash
aws rds create-db-subnet-group \
  --db-subnet-group-name "$DB_SUBNET_GROUP" \
  --db-subnet-group-description "Subredes para RDS - Tienda Master UCM" \
  --subnet-ids "$SUBNET_1" "$SUBNET_2"
```

Creamos la instancia. Usamos `--manage-master-user-password`, que hace que **RDS genere la contraseña y la guarde en Secrets Manager** automáticamente (no aparece en ningún sitio en texto plano):

```bash
aws rds create-db-instance \
  --db-instance-identifier "$DB_IDENTIFIER" \
  --engine postgres \
  --db-instance-class db.t3.micro \
  --allocated-storage 20 \
  --storage-type gp3 \
  --db-name shopdb \
  --master-username shopadmin \
  --manage-master-user-password \
  --db-subnet-group-name "$DB_SUBNET_GROUP" \
  --vpc-security-group-ids "$RDS_SG_ID" \
  --no-publicly-accessible \
  --backup-retention-period 0 \
  --no-multi-az
```

**Notas de los parámetros:**
- `db.t3.micro` + `20 GB gp3`: tamaño mínimo, elegible para *Free Tier* en muchas regiones.
- `--no-publicly-accessible`: la BD no tendrá IP pública; solo se llega desde dentro de la VPC.
- `--backup-retention-period 0` y `--no-multi-az`: desactivan copias y alta disponibilidad para abaratar el laboratorio. **En producción no se hace.**
- No fijamos `--engine-version`: RDS usa la versión por defecto de PostgreSQL disponible en tu región.

La creación tarda varios minutos. Espera a que esté lista:

```bash
aws rds wait db-instance-available --db-instance-identifier "$DB_IDENTIFIER"
```

Cuando termine, recarga las variables para capturar el *endpoint* y el ARN del secreto:

```bash
source setup-env.sh
echo "Endpoint BD: $DB_ENDPOINT"
echo "Secreto BD:  $DB_SECRET_ARN"
```

El secreto que crea RDS es un JSON con las claves `username` y `password`. En el **Bloque 2** inyectaremos esos dos campos como variables de entorno del contenedor directamente desde Secrets Manager (sin copiarlos a mano).

---

## Fase 4 — Backend (FastAPI) e imagen en ECR

### 4.1 Código de la aplicación

El backend es una API mínima en **FastAPI** con dos entidades (productos y pedidos). Al arrancar, crea las tablas y alimenta el catálogo de tecnología (así no necesitamos un paso de migración separado contra una BD privada). Si está configurada la variable `SNS_TOPIC_ARN`, publica un evento al crear un pedido — eso lo activaremos en el Bloque 3.

Crea la estructura del proyecto:

```bash
mkdir -p backend/app && cd backend
```

`app/main.py`:

```bash
cat > app/main.py << 'EOF'
import os, json, datetime
import boto3
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from sqlalchemy import create_engine, Column, Integer, String, Numeric, DateTime
from sqlalchemy.orm import declarative_base, sessionmaker

# --- Configuración desde variables de entorno (las inyecta ECS) ---
DB_HOST = os.environ["DB_HOST"]
DB_PORT = os.environ.get("DB_PORT", "5432")
DB_NAME = os.environ.get("DB_NAME", "shopdb")
DB_USER = os.environ["DB_USER"]
DB_PASS = os.environ["DB_PASS"]
SNS_TOPIC_ARN = os.environ.get("SNS_TOPIC_ARN", "")
REGION = os.environ.get("AWS_REGION", "eu-west-1")
MEDIA_BASE_URL = os.environ.get("MEDIA_BASE_URL", "")

DATABASE_URL = f"postgresql+psycopg://{DB_USER}:{DB_PASS}@{DB_HOST}:{DB_PORT}/{DB_NAME}"
engine = create_engine(DATABASE_URL, pool_pre_ping=True)
SessionLocal = sessionmaker(bind=engine)
Base = declarative_base()


class Product(Base):
    __tablename__ = "products"
    id = Column(Integer, primary_key=True)
    name = Column(String, nullable=False)
    description = Column(String, default="")
    price = Column(Numeric(10, 2), nullable=False)
    image = Column(String, default="")
    stock = Column(Integer, default=10)


class Order(Base):
    __tablename__ = "orders"
    id = Column(Integer, primary_key=True)
    email = Column(String, nullable=False)
    items = Column(String, nullable=False)
    total = Column(Numeric(10, 2), nullable=False)
    created_at = Column(DateTime, default=datetime.datetime.utcnow)


SEED = [
    {"name": "Portátil UCM Pro 14", "description": "Ultrabook 14\" 16GB RAM 512GB SSD", "price": 1199.00, "image": "laptop.jpg", "stock": 15},
    {"name": "Smartphone Campus X", "description": "6.5\" 128GB cámara triple", "price": 599.00, "image": "phone.jpg", "stock": 30},
    {"name": "Auriculares Aula Quiet", "description": "Cancelación de ruido, 30h batería", "price": 149.00, "image": "headphones.jpg", "stock": 50},
    {"name": "Monitor Master 27\"", "description": "QHD 144Hz IPS", "price": 329.00, "image": "monitor.jpg", "stock": 20},
    {"name": "Teclado Mecánico DXC", "description": "Switches marrones, retroiluminado", "price": 89.00, "image": "keyboard.jpg", "stock": 40},
    {"name": "Tablet Beca 11", "description": "11\" 256GB con lápiz", "price": 449.00, "image": "tablet.jpg", "stock": 18},
]

app = FastAPI(title="Tienda Master UCM")
app.add_middleware(
    CORSMiddleware, allow_origins=["*"], allow_methods=["*"], allow_headers=["*"]
)


@app.on_event("startup")
def init_db():
    Base.metadata.create_all(engine)
    db = SessionLocal()
    if db.query(Product).count() == 0:
        for p in SEED:
            db.add(Product(**p))
        db.commit()
    db.close()


@app.get("/health")
def health():
    return {"status": "ok"}


@app.get("/products")
def list_products():
    db = SessionLocal()
    rows = db.query(Product).all()
    db.close()
    return [
        {
            "id": p.id, "name": p.name, "description": p.description,
            "price": float(p.price), "stock": p.stock,
            "image": f"{MEDIA_BASE_URL}/{p.image}" if MEDIA_BASE_URL else p.image,
        }
        for p in rows
    ]


class OrderItem(BaseModel):
    product_id: int
    qty: int


class OrderIn(BaseModel):
    email: str
    items: list[OrderItem]


@app.post("/orders")
def create_order(order: OrderIn):
    db = SessionLocal()
    total, detail = 0.0, []
    for it in order.items:
        p = db.get(Product, it.product_id)
        if not p:
            db.close()
            raise HTTPException(404, f"Producto {it.product_id} no existe")
        total += float(p.price) * it.qty
        detail.append({"name": p.name, "qty": it.qty, "price": float(p.price)})
    o = Order(email=order.email, items=json.dumps(detail, ensure_ascii=False), total=total)
    db.add(o)
    db.commit()
    db.refresh(o)
    order_id = o.id
    db.close()

    if SNS_TOPIC_ARN:
        boto3.client("sns", region_name=REGION).publish(
            TopicArn=SNS_TOPIC_ARN,
            Subject="Nuevo pedido en Tienda Master UCM",
            Message=json.dumps(
                {"order_id": order_id, "email": order.email, "total": total, "items": detail},
                ensure_ascii=False,
            ),
        )
    return {"order_id": order_id, "total": total, "status": "creado"}
EOF
```

`requirements.txt`:

```bash
cat > requirements.txt << 'EOF'
fastapi==0.115.*
uvicorn[standard]==0.32.*
sqlalchemy==2.0.*
psycopg[binary]==3.2.*
boto3==1.35.*
EOF
```

`Dockerfile`:

```bash
cat > Dockerfile << 'EOF'
FROM python:3.12-slim
WORKDIR /srv
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app/ ./app/
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
EOF
```

### 4.2 Repositorio ECR

```bash
aws ecr create-repository --repository-name "$ECR_REPO" \
  --image-scanning-configuration scanOnPush=true
```

`scanOnPush=true` activa el escaneo de vulnerabilidades de la imagen al subirla — buena práctica que conviene mostrar a los alumnos.

### 4.3 Login, build y push

```bash
# Autenticar Docker contra tu registro ECR
aws ecr get-login-password --region "$AWS_REGION" \
  | docker login --username AWS --password-stdin \
    "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

# Construir la imagen (--platform por si compilas en Mac Apple Silicon)
docker build --platform linux/amd64 -t "$ECR_REPO:latest" .

# Etiquetar y empujar a ECR
docker tag "$ECR_REPO:latest" "$ECR_URI:latest"
docker push "$ECR_URI:latest"
```

`--platform linux/amd64` evita el error típico de quien construye en un Mac con chip ARM: Fargate (por defecto) ejecuta x86, y una imagen ARM no arrancaría.

Verifica que la imagen está en ECR:

```bash
aws ecr describe-images --repository-name "$ECR_REPO" \
  --query "imageDetails[].imageTags" --output text
```

Vuelve a la carpeta raíz de la práctica:

```bash
cd ..
```

---

## Checkpoint del Bloque 1

Antes de pasar al Bloque 2, comprueba que tienes todo:

```bash
source setup-env.sh
echo "VPC:         $VPC_ID"
echo "SG (alb/ecs/rds): $ALB_SG_ID / $ECS_SG_ID / $RDS_SG_ID"
echo "RDS endpoint: $DB_ENDPOINT"
echo "Secreto RDS:  $DB_SECRET_ARN"
echo "Imagen ECR:   $ECR_URI:latest"
```

Si las cinco líneas tienen valor (ninguna vacía ni `None`), el Bloque 1 está completo:

- [x] Variables y naming determinista (`setup-env.sh`)
- [x] Default VPC y subredes localizadas
- [x] Tres *security groups* encadenados (ALB → ECS → RDS)
- [x] RDS PostgreSQL disponible, con credenciales en Secrets Manager
- [x] Imagen del backend publicada en ECR


