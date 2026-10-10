# Laboratorio 1 — DevOps: Contenedores, CI/CD y Despliegue en AWS ECS/EC2

## 1. Arquitectura de contenedores y conceptos de CI/CD aplicados

### 1.1 Arquitectura de contenedores

La aplicación (`src/main.py`, FastAPI) se empaqueta como una **imagen Docker** inmutable
a partir del [`Dockerfile`](../Dockerfile):

```
python:3.12-slim
   └── poetry install --only main   (solo dependencias de producción)
   └── COPY src/ → /app/src
   └── ENTRYPOINT: uvicorn main:app --host 0.0.0.0 --port 8000
```

Esta imagen se publica en **Amazon ECR** (Elastic Container Registry), que actúa como
registro privado versionado: cada build genera dos tags, `:<git-sha>` (inmutable, para
trazabilidad) y `:latest` (apuntador móvil al último build exitoso).

El tiempo de ejecución (runtime) es **Amazon ECS (Elastic Container Service) sobre
EC2**, no Fargate, es decir, el clúster administra contenedores corriendo sobre
instancias EC2 propias:

```
┌─────────────────────────────── VPC por defecto ───────────────────────────────┐
│                                                                                │
│   Auto Scaling Group (asg-<suffix>)                                           │
│   └── EC2 Instance (t3.micro, AMI ECS-Optimized Amazon Linux 2023)            │
│         └── ECS Agent                                                        │
│               └── Contenedor Docker "greeting-app" (puerto 8000 → 8000)       │
│                                                                                │
│   Capacity Provider (cp-<suffix>) ── gestiona el escalado administrado        │
│   ECS Cluster (cluster-<suffix>) ── agrupa capacity provider + servicio       │
│   ECS Service (service-greeting-<suffix>) ── mantiene 1 tarea en ejecución     │
│   Security Group ── permite tráfico entrante TCP:8000 desde 0.0.0.0/0          │
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘
```

Componentes clave y su rol:

| Componente | Rol |
|---|---|
| **Launch Template** | Define cómo se lanza cada instancia EC2 (AMI, tipo de instancia, perfil IAM, *user data* que registra la instancia en el clúster ECS, volumen raíz). |
| **Auto Scaling Group (ASG)** | Garantiza que siempre haya la cantidad deseada de instancias EC2 (1 en este lab) disponibles para el clúster. |
| **Capacity Provider** | Conecta el ASG con el clúster ECS, permitiendo que ECS "sepa" dónde colocar tareas y realice *managed scaling*. |
| **Task Definition** | Plantilla (JSON) que describe el contenedor: imagen, CPU/memoria, puertos, variables, logging (`awslogs` hacia CloudWatch). |
| **ECS Service** | Mantiene corriendo el número deseado de tareas (réplicas) de una Task Definition y realiza *rolling deployments* cuando se actualiza. |
| **IAM Roles** | Un rol para la instancia EC2 (`AmazonEC2ContainerServiceforEC2Role`, permite que el agente ECS se comunique con el servicio) y un rol de ejecución de tarea (`AmazonECSTaskExecutionRolePolicy`, permite al contenedor hacer *pull* de ECR y escribir logs). |

Esta arquitectura permite separar claramente **la construcción del artefacto**
(imagen Docker versionada en ECR) de **su ejecución** (infraestructura ECS/EC2
aprovisionada y actualizada de forma declarativa/idempotente vía AWS CLI).

### 1.2 Conceptos de CI/CD aplicados

El pipeline se implementa con **GitHub Actions** y aplica los siguientes conceptos:

- **Integración Continua (CI):**
  - *Build automatizado y reproducible*: cada cambio en `main` (o PR) dispara `lint`
    (`pylint`) y `test` (`pytest`) sobre un entorno limpio (`ubuntu-latest`), usando
    **Poetry** para fijar versiones exactas de dependencias (`poetry.lock`).
  - *Fail-fast*: si `lint` o `test` fallan, el pipeline se detiene antes de construir
    o publicar ninguna imagen (`build_and_push` depende de `needs: [lint, test]`).
  - *Pull Requests como gate de calidad*: el workflow corre en `pull_request` hacia
    `main`, de modo que el código se valida antes de integrarse.

- **Entrega/Despliegue Continuo (CD):**
  - *Build de artefactos inmutables*: la imagen Docker se etiqueta con el `github.sha`,
    garantizando trazabilidad total entre un commit y la imagen desplegada.
  - *Registro centralizado de artefactos*: Amazon ECR como *container registry*.
  - *Despliegue automatizado e idempotente*: `scripts/deploy.sh` puede ejecutarse
    repetidamente sin duplicar recursos (verifica existencia antes de crear, y usa
    `update-service --force-new-deployment` para *rolling updates* sin downtime total).
  - *Infraestructura como código vía AWS CLI*: toda la infraestructura (red, IAM,
    cómputo, clúster, servicio) se declara y reconcilia mediante script versionado en
    el repositorio, no se crea manualmente en la consola.
  - *Separación de ambientes/alumnos*: la variable `STUDENT_SUFFIX` permite que cada
    estudiante tenga sus propios recursos aislados (naming namespacing) usando el
    **mismo pipeline**.
  - *Outputs encadenados entre jobs*: `build_and_push` expone `image_uri` como
    `output`, consumido por `deploy` vía `needs.build_and_push.outputs.image_uri`,
    evitando *hardcodear* la imagen a desplegar.
  - *Triggers controlados*: `build_and_push` y `deploy` solo corren en `push` a `main`
    o `workflow_dispatch` manual — nunca en PRs — evitando desplegar código no
    integrado.
  - *Operación de rollback/limpieza como pipeline independiente*: el workflow
    [`destroy.yml`](../.github/workflows/destroy.yml) implementa el concepto de
    **pipeline bajo demanda** (`workflow_dispatch` only) para destruir el ambiente,
    separando intencionalmente el ciclo de vida de "crear" del de "destruir" y
    añadiendo una confirmación manual (`confirm: destroy`) como control de seguridad.

## 2. Evidencia visual de la ejecución exitosa del pipeline de GitHub Actions

> ⚠️ **Pendiente de completar con capturas reales.** Ejecuta los workflows en tu
> repositorio de GitHub, descarga o captura las pantallas correspondientes y
> guárdalas en [`docs/evidence/`](./evidence) con los nombres sugeridos abajo.
> Luego las imágenes se mostrarán automáticamente en este documento gracias a la
> sintaxis Markdown ya dejada lista (`![descripción](ruta)`).

### 2.1 Pipeline automático para desplegar (`.github/workflows/ci.yml`)

Pasos sugeridos para capturar la evidencia:
1. Haz push a `main` (o dispara `workflow_dispatch` en **CI/CD Pipeline (Lab 1)**).
2. En la pestaña **Actions**, abre la ejecución y espera a que los 4 jobs terminen en OK.
3. Captura: (a) la vista general de la ejecución con todos los jobs en verde, y
   (b) el log del job `build_and_push` mostrando el `docker push` exitoso y el
   log del job `deploy` mostrando la URL pública final impresa por `deploy.sh`.

```markdown
![Pipeline CI/CD completo en verde](./evidence/ci-pipeline-success.png)

![Logs de build_and_push y deploy](./evidence/ci-pipeline-deploy-logs.png)
```

### 2.2 Pipeline manual para eliminar infraestructura (`.github/workflows/destroy.yml`)

Pasos sugeridos para capturar la evidencia:
1. En **Actions → Destroy Infrastructure (Lab 1) → Run workflow**, ingresa el
   `student_suffix` y escribe `destroy` en el campo `confirm`.
2. Espera a que el job `destroy` finalice en OK.
3. Captura: (a) el formulario de `workflow_dispatch` con los inputs diligenciados
   antes de ejecutar, y (b) el resultado final del job mostrando la limpieza de
   cada recurso (ECS, ASG, Security Group, IAM, ECR, logs).

```markdown
![Formulario de ejecución manual de destroy.yml](./evidence/destroy-pipeline-inputs.png)

![Ejecución exitosa de destroy.yml](./evidence/destroy-pipeline-success.png)
```

## 3. Evidencia visual de la petición HTTP a la IP pública de la instancia EC2

> ⚠️ **Pendiente de completar con capturas reales.** Tras un despliegue exitoso,
> `deploy.sh` imprime en el log la IP pública y las URLs de prueba
> (`/`, `/health`, saludo personalizado). Usa esa IP con Postman, `curl` o el
> navegador y guarda la evidencia en [`docs/evidence/`](./evidence).

Ejemplos de comandos/consultas a documentar:

```bash
# Saludo personalizado
curl "http://<IP_PUBLICA>:8000/?first_name=John&last_name=Doe"

# Health check
curl "http://<IP_PUBLICA>:8000/health"
```

```markdown
![Respuesta HTTP del endpoint de saludo (Postman/curl/navegador)](./evidence/http-greeting-response.png)

![Respuesta HTTP del endpoint /health](./evidence/http-health-response.png)
```

## 4. Propuestas para mejorar el pipeline

1. **Escaneo de seguridad de la imagen (SCA/SAST):** integrar `trivy` o
   `docker scout` como job adicional antes de `build_and_push`, bloqueando el
   pipeline si se detectan vulnerabilidades críticas en la imagen o dependencias.
2. **Cobertura de pruebas y calidad de código:** agregar `pytest --cov` con un
   umbral mínimo de cobertura y subir el reporte como artifact o a un servicio
   como Codecov; incorporar `black`/`ruff` además de `pylint`.
3. **Entornos (environments) de GitHub con aprobación manual:** usar
   `environment: production` en el job `deploy` para requerir aprobación humana
   antes de desplegar a AWS, añadiendo un *gate* de control de cambios.
4. **Versionado semántico e inmutabilidad total de imágenes:** además del `sha`,
   generar tags semánticos (`v1.2.3`) a partir de *git tags*, y evitar
   sobrescribir `:latest` en paralelo (actualmente dos pushes concurrentes podrían
   generar una condición de carrera sobre ese tag).
5. **Cache de dependencias:** usar `actions/cache` para el entorno virtual de
   Poetry y las capas de Docker (`docker/build-push-action` con `cache-from`/
   `cache-to` en ECR o GitHub Actions cache), reduciendo tiempos de `lint`,
   `test` y `build_and_push`.
6. **Infraestructura como código declarativa:** migrar `deploy.sh`/`destroy.sh`
   (imperativos, basados en AWS CLI) a **Terraform** o **AWS CDK**, obteniendo
   *plan/apply* con revisión de cambios, estado remoto y *drift detection*.
7. **Alta disponibilidad y balanceo de carga:** reemplazar la IP pública directa
   de una única instancia EC2 por un **Application Load Balancer** + múltiples
   instancias en distintas zonas de disponibilidad, eliminando el punto único de
   fallo y permitiendo despliegues *blue/green* o *canary*.
8. **Notificaciones del pipeline:** agregar un paso final que notifique el
   resultado (éxito/fallo) del pipeline a Slack/Teams/correo, incluyendo la URL
   desplegada, en lugar de depender de revisar manualmente los logs de Actions.
9. **Pruebas de humo post-despliegue:** tras `deploy`, agregar un step que haga
   `curl` automático a `/health` y al endpoint de saludo, fallando el pipeline
   (y opcionalmente disparando rollback) si la aplicación no responde `200`.
10. **Gestión de secretos y mínimo privilegio:** sustituir las credenciales IAM
    de usuario (`AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`) por **OIDC entre
    GitHub Actions y AWS** (`aws-actions/configure-aws-credentials` con
    `role-to-assume`), eliminando claves de larga duración almacenadas como secret.
