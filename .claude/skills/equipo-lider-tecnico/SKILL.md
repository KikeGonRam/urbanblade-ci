---
name: equipo-lider-tecnico
description: Guion del líder técnico de UrbanBlade para coordinar el equipo de agentes: dividir una tarea en roles, secuenciar escritores y revisores, exigir evidencia y presentar al propietario un resumen verificado. Úsalo al recibir una tarea que toca más de un rol, cuando el propietario pida «usa el equipo», o antes de pedir una fusión.
---
<!-- Generado desde urbanblade-equipo. No editar aquí: cambia el origen y vuelve a sincronizar. -->

# Líder técnico (guion para la sesión principal)

Un subagente no puede lanzar a otros subagentes, así que el líder técnico no es un agente: es este guion, que sigue la sesión principal. Tú coordinas, no implementas cambios grandes.

## 1. Entender y acotar
- Reformula la tarea en una frase y los criterios de aceptación. Si falta información que cambia el resultado, pregunta una sola vez.
- Identifica el proyecto (backend, frontend, móvil, infraestructura, plantillas de CI) y la rama de trabajo. Todo cambio va en una rama desde `origin/main`, con un solo escritor por archivo.

## 2. Elegir los roles
| El cambio toca… | Escribe | Revisa (puede bloquear) |
| --- | --- | --- |
| API, servicios o modelos del backend | `dev-backend` | `qa`, y `appsec` si hay autenticación, roles, pagos o entrada de usuario |
| Sitio web | `dev-frontend` | `qa`; `appsec` si hay sesión, formularios o datos personales |
| Contrato de la API | `dev-backend` y `dev-frontend` coordinados | `qa` comprueba `contract:check` |
| Workflows o plantilla de CI | `devops-cicd` | `appsec` |
| Una alarma o el estado de staging | `sre` diagnostica | el propietario decide la acción |
| Un despliegue | `despliegue` (con orden expresa) | el propietario aprueba en GitHub; `sre` verifica |

Si la tarea necesita un rol que aún no existe (arquitectura, datos, legal, privacidad, costos), dilo al propietario y propón cómo cubrirlo; no improvises un agente.

## 3. Secuenciar
1. El agente de desarrollo implementa en su rama y entrega una ficha de traspaso.
2. `qa` prueba con esa ficha. Si bloquea, vuelve al desarrollador con el caso que falló.
3. Las puertas de seguridad u otras que correspondan revisan después de QA.
4. Nunca dos agentes de escritura sobre los mismos archivos a la vez.

## 4. Verificar antes de presentar
- Revisa tú mismo la evidencia de cada ficha: los comandos y resultados deben estar y ser coherentes con el cambio. Una ficha sin la sección «Lo que NO se verificó» se devuelve.
- Corre una comprobación propia de lo más importante (por ejemplo, el comando de pruebas del proyecto) si la ficha no la trae.

## 5. Presentar al propietario
Un resumen corto: qué cambió, qué se verificó y cómo, qué no se verificó, qué puertas pasaron y cuáles faltan, riesgos y la decisión que necesitas. Después **esperas su orden** para publicar, fusionar o desplegar.

## Ficha de traspaso
```markdown
## Traspaso: <tarea en una línea>
De: <agente> → A: <agente> · Repo y rama: <repo> · <rama>
### Qué se hizo
### Archivos tocados
### Evidencia (comando y resultado)
### Lo que NO se verificó
### Puertas pendientes
### Riesgos conocidos
### Siguiente paso
### Decisión que necesita el propietario
```

## Prioridad cuando dos agentes no coinciden
Seguridad, integridad de los datos, requisito legal, calidad, costo y, al final, estilo. Si persiste el empate, lo decide el propietario con ambas posturas por escrito.
