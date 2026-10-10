# DevOps Lab 1 — Greeting App

Aplicación **FastAPI** de ejemplo para el Laboratorio 1 de DevOps (UNAL). Expone un
endpoint que recibe un nombre y un apellido y devuelve un saludo personalizado, junto
con un endpoint de *health check*. El proyecto incluye pipeline de **CI/CD con GitHub
Actions** que compila, prueba, construye la imagen Docker, la publica en **Amazon ECR**
y despliega el servicio en un clúster **ECS sobre EC2**.

## Contenido del proyecto

```
devops-lab/
├── src/
│   ├── main.py                 # Aplicación FastAPI (endpoints / y /health)
│   └── function_greeting/
│       └── main.py             # Variante como AWS Lambda handler (lambda_handler)
├── tests/
│   └── test_greeting.py        # Pruebas unitarias con pytest + TestClient
├── scripts/
│   └── deploy.sh               # Script idempotente de despliegue a ECS/EC2 vía AWS CLI
├── workflows/
│   └── ci.yml                  # Definición del pipeline CI/CD (GitHub Actions)
├── Dockerfile                  # Imagen de contenedor para la app FastAPI
├── pyproject.toml              # Dependencias y configuración (Poetry, pylint)
├── poetry.lock                 # Lockfile de dependencias
├── .dockerignore / .gitignore
└── devops-lab1-starter.zip     # Material base/plantilla original del laboratorio
```

> **Nota:** el archivo de workflow vive en [`workflows/ci.yml`](./workflows/ci.yml). Para
> que GitHub Actions lo ejecute automáticamente debe copiarse (o enlazarse) a la ruta
> estándar `.github/workflows/ci.yml` del repositorio.

## Aplicación

La API (`src/main.py`) expone:

| Método | Ruta | Descripción |
|--------|------|-------------|
| `GET` | `/` | Devuelve `{"message": "Hello, {first_name} {last_name}!"}` si se envían los query params `first_name` y `last_name`; de lo contrario un mensaje genérico. |
| `GET` | `/health` | Devuelve `{"status": "healthy"}` para chequeos de salud (ECS/ALB). |

También existe una variante equivalente como función Lambda (`src/function_greeting/main.py`)
con la misma lógica de saludo, pensada para despliegue *serverless*.

## Requisitos

- Python 3.12
- [Poetry](https://python-poetry.org/) para gestión de dependencias
- Docker (para construir/ejecutar la imagen)
- AWS CLI configurado (solo necesario para build/push/deploy en ECR/ECS)

## Desarrollo local

```bash
# Instalar dependencias (incluye grupo dev: pylint, pytest, httpx)
poetry install

# Ejecutar la aplicación localmente
poetry run uvicorn main:app --reload --app-dir src --port 8000

# Linting
poetry run pylint src

# Pruebas unitarias
poetry run pytest tests -vv
```

La API quedará disponible en `http://localhost:8000`, por ejemplo:
`http://localhost:8000/?first_name=John&last_name=Doe`.

## Docker

```bash
docker build -t greeting-app .
docker run -p 8000:8000 greeting-app
```

El `Dockerfile` usa `python:3.12-slim`, instala solo dependencias de producción con
Poetry (`--only main`) y expone el puerto `8000` ejecutando `uvicorn`.

## Pipeline CI/CD ([`workflows/ci.yml`](./workflows/ci.yml))

Se dispara en `push`/`pull_request` a `main` y manualmente vía `workflow_dispatch`
(permite indicar `student_suffix` para namespacing de recursos AWS). Jobs:

1. **`lint`** — Instala dependencias con Poetry y corre `pylint` sobre `src`.
2. **`test`** — Instala dependencias y corre `pytest` sobre `tests`.
3. **`build_and_push`** *(solo en push a `main` o `workflow_dispatch`)* — Construye la
   imagen Docker, crea el repositorio ECR si no existe, y publica las etiquetas
   `:{sha}` y `:latest`, exponiendo la URI de imagen como output (`image_uri`).
4. **`deploy`** *(depende de `build_and_push`)* — Ejecuta `scripts/deploy.sh` con las
   variables `STUDENT_SUFFIX`, `AWS_REGION` e `IMAGE_URI` para desplegar/actualizar el
   servicio en ECS.

Variables/secrets usados por el pipeline:

- `vars.AWS_REGION`, `vars.STUDENT_SUFFIX` (variables de repositorio, con defaults
  `us-east-1` / `test-student`)
- `secrets.AWS_ACCESS_KEY_ID`, `secrets.AWS_SECRET_ACCESS_KEY` (credenciales AWS)

## Script de despliegue ([`scripts/deploy.sh`](./scripts/deploy.sh))

Script Bash idempotente (`set -euo pipefail`) que aprovisiona y actualiza toda la
infraestructura necesaria en AWS para servir la aplicación:

- Grupo de seguridad con acceso al puerto `8000`.
- Roles IAM para instancias EC2 y para la ejecución de tareas ECS.
- Clúster ECS, AMI ECS-optimizada (Amazon Linux 2023), Launch Template y Auto Scaling
  Group (1 instancia `t3.micro` por defecto).
- Capacity Provider administrado asociado al clúster.
- Registro de la definición de tarea (Task Definition) con la imagen publicada en ECR.
- Creación o actualización (`force-new-deployment`) del servicio ECS.
- Al finalizar, localiza la IP pública de la instancia y muestra las URLs de prueba
  (`/`, `/health`, saludo personalizado).

Variables de entorno relevantes (todas opcionales salvo `STUDENT_SUFFIX`):

| Variable | Descripción | Default |
|----------|-------------|---------|
| `STUDENT_SUFFIX` | Sufijo para namespacing de todos los recursos AWS | *(requerido)* |
| `APP_NAME` | Nombre base de la aplicación | `greeting` |
| `AWS_REGION` | Región AWS | `us-east-1` |
| `INSTANCE_TYPE` | Tipo de instancia EC2 | `t3.micro` |
| `ROOT_VOLUME_SIZE` | Tamaño del volumen raíz (GiB) | `30` |
| `IMAGE_TAG` | Tag de imagen a desplegar | `latest` |
| `IMAGE_URI` | URI completa de la imagen (si se omite, se calcula desde ECR) | *(calculado)* |

Ejecución manual de ejemplo:

```bash
STUDENT_SUFFIX=mi-usuario AWS_REGION=us-east-1 bash scripts/deploy.sh
```

## Pruebas

Las pruebas (`tests/test_greeting.py`) usan `fastapi.testclient.TestClient` y cubren:

- Saludo personalizado con `first_name` y `last_name`.
- Mensaje genérico sin parámetros.
- Endpoint `/health`.

```bash
poetry run pytest tests -vv
```
