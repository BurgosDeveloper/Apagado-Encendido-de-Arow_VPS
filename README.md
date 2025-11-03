# Automatización de encendido/apagado Arow VPS (EC2 + RDS + Docker/EasyPanel)

Este repo orquesta con GitHub Actions el encendido y apagado de:
- EC2 (con Docker y EasyPanel + servicios como web, n8n, chatwoot)
- RDS (PostgreSQL/MySQL según tu instancia)

## Horarios y orden

- Encendido: 07:00 VET (11:00 UTC) de lunes a sábado. Domingo no enciende.
  - Orden: EC2 -> (vía SSM: asegurar Docker y contenedores arriba) -> RDS (al final).
- Apagado: 12:00 VET (16:00 UTC) de lunes a sábado.
  - Orden: contenedores Docker (vía SSM) -> EC2 -> RDS (al final).

Los workflows están en `.github/workflows`:
- `start-infra.yml`: arranca EC2, levanta Docker/servicios por SSM, luego RDS y hace healthchecks HTTP.
- `stop-infra.yml`: detiene contenedores por SSM, apaga EC2 y al final detiene RDS.

## Requisitos

1. AWS SSM (agente y permisos):
   - La instancia EC2 debe tener el SSM Agent instalado y corriendo.
   - La IAM principal usada por GitHub Actions debe tener permisos para `ssm:SendCommand`, `ssm:GetCommandInvocation` además de EC2 y RDS.
2. Secrets en el repositorio:
   - `AWS_REGION` (por ejemplo, `us-east-1`)
   - `AWS_ACCESS_KEY_ID`
   - `AWS_SECRET_ACCESS_KEY`
3. Variables opcionales (Settings > Variables):
   - `EASYPANEL_URL`: URL del panel para healthcheck (opcional).
   - `HEALTHCHECK_URLS`: una o más URLs (una por línea) para verificar web/n8n/chatwoot, etc. (opcional).
   - `CONTAINER_NAMES`: nombres esperados de contenedores (espacio separado) para verificación adicional por SSM (opcional), por ejemplo:
     ```
     easypanel n8n chatwoot web
     ```

## Detalles de verificación

- Healthchecks HTTP: si se configuran `EASYPANEL_URL` y/o `HEALTHCHECK_URLS`, se intenta hasta 30 veces con 10s entre intentos. Cuenta como OK si retorna HTTP 2xx/3xx.
- Verificación Docker via SSM:
  - Inicio: arranca el servicio `docker` si es necesario y hace `docker start` sobre todos los contenedores existentes. Si `CONTAINER_NAMES` está definido, verifica que estén arriba por nombre.
  - Apagado: detiene todos los contenedores en ejecución (`docker stop`).

## Ejecución manual

Puedes disparar manualmente ambos workflows desde la pestaña "Actions" con `Run workflow` (usa `workflow_dispatch`).

## Notas y resolución de problemas

- Si los pasos SSM fallan, verifica:
  - EC2 tiene rol/permisos para SSM y el agente está en estado `Online` en Systems Manager.
  - La política de IAM de los secretos usados en Actions incluye permisos SSM.
- Si algún servicio depende de RDS para iniciar, considera ajustar la lógica para levantar RDS antes o en paralelo, y mantener healthchecks hasta que todo responda. Actualmente se cumple el requisito de "RDS al final".
- Ajusta las URLs de healthcheck a endpoints que no requieran autenticación (por ejemplo `/healthz`).
