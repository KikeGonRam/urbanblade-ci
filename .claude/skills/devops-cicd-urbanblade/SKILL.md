---
name: devops-cicd-urbanblade
description: Contexto y lecciones del pipeline de UrbanBlade: workflows de barber y frontend, la plantilla urbanblade-ci, OIDC con AWS, protecciones de rama y trampas ya vividas. Úsalo antes de modificar un workflow de GitHub Actions o la plantilla de despliegue.
---
<!-- Generado desde urbanblade-equipo. No editar aquí: cambia el origen y vuelve a sincronizar. -->

# CI/CD de UrbanBlade

## Piezas
- **Plantilla compartida:** `KikeGonRam/urbanblade-ci`, workflow `.github/workflows/desplegar-ecs.yml`. Barber y frontend la llaman desde su `desplegar-staging.yml`, **fijada por SHA de commit**, con parámetros (ECR, servicio de ECS, URL, rutas de salud, Dockerfile, migraciones, script de humo). Un cambio en la plantilla llega a un repo solo con un PR que actualiza ese SHA. Su `main` exige PR y el check `actionlint`.
- **Pipeline de despliegue:** arranca tras el CI de `main` (`workflow_run`) o a mano; decide si el commit cambia la imagen; exige el CI en verde del commit exacto; espera la aprobación del entorno `staging`; asume AWS por OIDC; respalda la imagen como `rollback-<fecha>-pre-<sha>`; construye, sube, redespliega, verifica salud, corre `.github/scripts/humo.sh` y revierte solo si algo falla.
- **AWS:** rol `urbanblade-staging-github-deploy`, definido en Terraform (`oidc_despliegue.tf`, repo `barberterraform`). Su confianza usa el `sub` inmutable del repo y el entorno `staging`.
- **Protecciones de `main`:** PR obligatorio, checks verdes (10 en barber, 11 en frontend), conversaciones resueltas, incluidos administradores, sin push forzado.

## Reglas al editar workflows
- Permisos mínimos por job; `permissions: {}` a nivel de workflow.
- Acciones de terceros fijadas por SHA con la versión en un comentario; sin entradas no confiables interpoladas en `run` (pásalas por `env`).
- Valida con `actionlint` (imagen `rhysd/actionlint`) antes de entregar. Un workflow no se da por probado hasta que corre en GitHub.
- La lógica común va en la plantilla; lo específico de cada repo (`humo.sh`, patrones de archivos, migraciones) va en el repo.

## Trampas ya vividas
1. **La variable del rol de AWS** (`AWS_DEPLOY_ROLE_ARN`) estaba solo en el entorno `staging`: el job que llama a una plantilla no tiene entorno y la veía vacía. Ahora existe también como variable de repositorio. Cualquier valor que lea el job llamador debe estar a nivel de repo.
2. **Aprobar antes de que termine el CI:** el despliegue se niega con «El CI de <sha> no terminó en verde». No es un fallo; hay que esperar al CI y relanzar.
3. **Un script sin `jq` local:** las pruebas de humo usan `jq` (el runner lo trae); para probarlas en local usa un contenedor `alpine` con `curl`, `jq` y `bash`.
4. **El dominio del frontend** a veces no responde desde la red local de desarrollo; los runners de GitHub sí lo alcanzan.
5. **`shellcheck`:** `A && B || error` es intencional en los scripts de humo y lleva su directiva `disable=SC2015` antes del primer comando.

## Git y publicación
Rama nueva desde `origin/main`; commits y PR en español; el repo del backend no lleva atribuciones de herramientas. Nada se publica ni se fusiona sin la orden del propietario. Tras un cambio al flujo de despliegue, propón un despliegue manual de prueba, que el propietario aprueba.
