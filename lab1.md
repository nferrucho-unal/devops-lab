# Laboratorio 1: arquitectura de contenedores y CI/CD

El proyecto consiste en un endpoint que recibe un nombre y un apellido y devuelve un saludo personalizado, junto con un endpoint de health check. El proyecto incluye pipeline de CI/CD con GitHub Actions que compila, prueba, construye la imagen Docker, la publica en Amazon ECR y despliega el servicio en un clúster ECS sobre EC2. 

## Estructura del proyecto

```text
devops-lab/
├── src/
│   ├── main.py                 # Aplicación FastAPI (endpoints / y /health)
│   └── function_greeting/
│       └── main.py             # Variante como AWS Lambda handler (lambda_handler)
├── tests/
│   └── test_greeting.py        # Pruebas unitarias con pytest + TestClient
├── scripts/
│   ├── deploy.sh                # Script idempotente de despliegue a ECS/EC2 vía AWS CLI
│   └── destroy.sh               # Script idempotente de destrucción de toda la infraestructura
├── workflows/
│   └── ci.yml                  # Definición del pipeline CI/CD (GitHub Actions)
├── .github/
│   └── workflows/
│       └── destroy.yml         # Workflow manual (workflow_dispatch) para destruir infraestructura
├── Dockerfile                  # Imagen de contenedor para la app FastAPI
├── pyproject.toml              # Dependencias y configuración (Poetry, pylint)
├── poetry.lock                 # Lockfile de dependencias
├── .dockerignore / .gitignore
└── devops-lab1-starter.zip     # Material base/plantilla original del laboratorio
```

## Arquitectura y orquestación de componentes

La aplicación FastAPI se empaqueta en una imagen Docker con Python 3.12 y sus dependencias. La imagen se publica en Amazon ECR y ECS la ejecuta en una instancia EC2.


ECS usa una Task Definition para especificar la imagen y la configuración del contenedor. Un ECS Service mantiene una tarea ejecutándose. La instancia EC2 pertenece a un Auto Scaling Group, configurado en este laboratorio para mantener una instancia. 

## CI/CD

El workflow de GitHub Actions automatiza la validación y el despliegue:

En pull requests y cambios en main, ejecuta Pylint y Pytest.
Si ambas validaciones pasan, un push a main construye la imagen y la pública en ECR.
La imagen se etiqueta con el SHA del commit para identificar su versión.
Después, el job de despliegue ejecuta deploy.sh para actualizar el servicio ECS.

## Evidencias

### Ejecución del pipeline de GitHub Actions

Ejecución CI/CD pipeline

![Texto alternativo] /images/image1.pic

El pipeline de GitHub Actions valida los cambios con Pylint y Pytest. En un push a main, si ambas comprobaciones pasan, construye la imagen Docker, la publica en Amazon ECR y despliega la aplicación en ECS sobre EC2. Cada imagen queda identificada por el SHA del commit. Los pull requests solo ejecutan las validaciones; no despliegan.



### Ejecución de Destroy.yml

Pendiente parámetros para ejecución procesos de eliminación

In Execution…



### Petición HTTP a la IP pública de EC2




## Propuestas para mejorar el pipeline

Agregar pruebas posteriores al despliegue: consultar /health y comprobar que responda correctamente antes de marcar el pipeline como exitoso.
Reducir tiempos de ejecución: habilitar caché de dependencias de Poetry y de capas Docker.
Evitar depender de latest: desplegar únicamente la imagen identificada por SHA del commit y conservar una versión anterior para facilitar el rollback.

## Comparación de workflows

ci-pruebas.yml es una variante manual para probar mejoras; no reemplaza ci.yml. Ambos ejecutan lint y pruebas, construyen una imagen y pueden desplegarla en ECS.


| Aspecto | `ci.yml` | `ci-pruebas.yml` |
|---|---|---|
| Cuándo corre | En push o PR hacia main; también se puede ejecutar manualmente. | Solo al iniciarlo manualmente desde Actions. |
| Poetry | Lo instala después de setup-python. | Lo instala antes de setup-python, necesario para que funcione cache: poetry. |
| Imagen Docker | Publica etiquetas SHA y latest, sin caché de capas configurada. | Publica solo la etiqueta SHA y usa caché de capas Docker. |
| ECR | No configura retención de imágenes. | Conserva las 10 imágenes etiquetadas más recientes para facilitar rollback. |
| Después del despliegue | Ejecuta deploy.sh. | Ejecuta deploy.sh y consulta /health; si no responde saludable, el workflow falla. |

Importante: ci-pruebas.yml también despliega en AWS cuando lo ejecutas manualmente; no es solo un workflow de pruebas unitarias.
