# urbanblade-ci

Plantillas de GitHub Actions compartidas por los repositorios de UrbanBlade. Hoy tiene una: el despliegue controlado de un servicio a AWS ECS Fargate. Solo contiene workflows: ningún secreto, credencial ni dato.

## `desplegar-ecs.yml`

Despliega una imagen de Docker a un servicio de ECS con aprobación, reversión automática y pruebas de humo. Lo usan `KikeGonRam/barber` y `KikeGonRam/frontend_Urbanblade`.

Qué hace, en orden:
1. **Decidir:** si el commit solo cambia documentación, workflows, pruebas o archivos de agentes, no despliega ni pide aprobación. Si trae migraciones o seeders (cuando el repositorio los declara), no despliega solo.
2. **Proteger:** comprueba que el CI de ese commit exacto terminó en verde y que el origen es un push a `main` del propio repositorio.
3. **Aprobación:** el entorno de GitHub (`staging`) exige la revisión del propietario.
4. **AWS por OIDC:** asume un rol de privilegio mínimo; no hay llaves de larga duración.
5. **Respaldo y despliegue:** etiqueta la imagen actual como `rollback-<fecha>-pre-<sha>`, construye, sube con `sha-<corto>` y `latest`, y redespliega.
6. **Verificar:** el digest de las tareas debe ser el recién subido y las rutas de salud deben responder 200.
7. **Pruebas de humo:** ejecuta el script que define cada repositorio.
8. **Revertir:** si algo de lo anterior falla, vuelve a la imagen anterior.

### Cómo se llama

El repositorio que llama define los disparadores, la concurrencia y los permisos del job, que deben incluir `contents: read`, `actions: read` e `id-token: write`. Siempre se fija por SHA de commit, nunca por rama.

```yaml
name: Desplegar a staging

on:
  workflow_run:
    workflows: [CI]
    types: [completed]
    branches: [main]
  workflow_dispatch:
    inputs:
      confirmar_migraciones:
        type: boolean
        default: false

permissions: {}

concurrency:
  group: desplegar-staging-barber
  cancel-in-progress: false

jobs:
  desplegar:
    uses: KikeGonRam/urbanblade-ci/.github/workflows/desplegar-ecs.yml@<sha-de-commit>
    permissions:
      contents: read
      actions: read
      id-token: write
    with:
      ecr_repository: urbanblade/barber
      ecs_service: uba-stg-barber
      ecs_container: barber
      health_url: https://api.urbanblade.com.mx
      environment_url: https://api.urbanblade.com.mx/up
      rutas_salud: /up /api/v1/barbershop
      role_arn: ${{ vars.AWS_DEPLOY_ROLE_ARN }}
      dockerfile: .docker/staging/Dockerfile
      rutas_migraciones: database/migrations database/seeders
      confirmar_migraciones: ${{ inputs.confirmar_migraciones || false }}
      script_humo: .github/scripts/humo.sh
```

### Entradas

| Entrada | Obligatoria | Por defecto | Para qué sirve |
| --- | --- | --- | --- |
| `ecr_repository`, `ecs_service`, `ecs_container` | Sí | | Recursos de AWS que se despliegan. |
| `health_url`, `environment_url`, `rutas_salud` | Sí | | URL pública y rutas que deben responder 200. |
| `role_arn` | Sí | | Rol de AWS que se asume por OIDC. |
| `ecs_cluster`, `aws_region`, `environment` | No | `urbanblade-staging`, `us-east-1`, `staging` | Destino y entorno de aprobación. |
| `dockerfile` | No | `Dockerfile` | Archivo de construcción. |
| `ci_workflow` | No | `ci.yml` | Workflow de CI cuyo verde se exige. |
| `patron_sin_imagen` | No | documentación, `.github/`, `tests/`, agentes | Qué archivos no cambian la imagen (`grep -E`). |
| `rutas_migraciones`, `confirmar_migraciones` | No | vacío, `false` | Si hay migraciones, el despliegue se detiene salvo confirmación. |
| `script_humo` | No | vacío | Script de pruebas de humo del repositorio, de solo lectura. |

### Seguridad

- El rol de AWS confía en el repositorio que llama y en el entorno (`sub` del token OIDC), no en este repositorio. Cambiar esta plantilla no da acceso a AWS por sí solo.
- Los repositorios que llaman fijan la versión por SHA: un cambio aquí no llega a un despliegue sin un PR en cada uno.
- Este repositorio debe tener `main` protegida con PR obligatorio, como los demás.
