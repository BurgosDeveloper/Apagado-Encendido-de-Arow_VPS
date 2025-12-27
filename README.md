# Automatización de encendido/apagado Arow VPS (EC2 + RDS)

Este repo orquesta con GitHub Actions el encendido y apagado de:
- EC2
- RDS (PostgreSQL/MySQL según tu instancia)

## Horarios y orden

- Encendido: 07:00 VET (11:00 UTC) de lunes a sábado. Domingo no enciende.
  - Orden: EC2 -> RDS.
- Apagado: 00:00 VET (04:00 UTC) diario (medianoche).
  - Orden: EC2 -> RDS.

Los workflows están en `.github/workflows`:
- `start-infra.yml`: arranca EC2, espera checks 2/2 y luego arranca RDS.
- `stop-infra.yml`: detiene EC2 y luego detiene RDS.

## Requisitos

1. Secrets en el repositorio:
   - `AWS_REGION` (por ejemplo, `us-east-1`)
   - `AWS_ACCESS_KEY_ID`
   - `AWS_SECRET_ACCESS_KEY`

La IAM asociada a esas credenciales debe tener permisos para EC2 y RDS (start/stop/wait/describe).

## Detalles de verificación

Los workflows hacen una verificación final consultando estados:
- EC2: `describe-instances` -> `State.Name`
- RDS: `describe-db-instances` -> `DBInstanceStatus`

## Ejecución manual

Puedes disparar manualmente ambos workflows desde la pestaña "Actions" con `Run workflow` (usa `workflow_dispatch`).

Prueba recomendada (manual):
1. Ejecuta “Detener EC2 y RDS” y valida que EC2 queda `stopped` y RDS queda `stopped`.
2. Ejecuta “Arrancar EC2 y RDS” y valida que EC2 queda `running` y RDS queda `available`.

## Notas y resolución de problemas

- Si algún servicio depende de RDS para iniciar, este repo no valida servicios internos; solo garantiza encendido/apagado de EC2 y RDS.
