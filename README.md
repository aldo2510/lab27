# Lab 27: Monitoreo de Pipeline con Jenkins + Prometheus + Pushgateway (+ Alertmanager)

> **Grafana está en otro lado**: usa tu instancia existente de Grafana y apúntala al Prometheus que levantarás aquí.

## Estructura
```
lab27/
├── Jenkinsfile
├── app/
│   ├── app.js
│   ├── package.json
│   └── Dockerfile
├── monitoring/
│   ├── docker-compose.yml
│   ├── prometheus/
│   │   ├── alerts.yml
│   │   └── prometheus.yml
│   └── alertmanager/
│       └── alertmanager.yml
└── grafana/
    └── grafana-lab27-dashboard.json
```

## Requisitos
- Jenkins funcionando.
- Docker y Docker Compose en el host donde correrá `monitoring/`.
- Jenkins con **Prometheus metrics plugin** habilitado (endpoint `/prometheus`).

## 1) Levantar Prometheus + Pushgateway (+ Alertmanager)
```bash
cd monitoring
# Edita prometheus/prometheus.yml y cambia jenkins-host:8080 por la dirección real
docker compose up -d
# Prometheus:  http://<este-host>:9090
# Pushgateway: http://<este-host>:9091
# Alertmanager: http://<este-host>:9093
```

> **Importante**: desde Jenkins debe poderse llegar a `http://<este-host>:9091` (Pushgateway).
> Desde Grafana, debe poderse llegar a `http://<este-host>:9090` (Prometheus).

## 2) Configurar Jenkins
- Instala y habilita el **Prometheus metrics plugin**.
- Crea un job *Pipeline (o Multibranch)* apuntando a este repo/carpeta.
- En el `Jenkinsfile`, reemplaza `PUSHGATEWAY_URL` por tu `http://<pushgateway-host>:9091`.

## 3) Ejecutar el pipeline
- El pipeline:
  - Envía métricas a Pushgateway al inicio y al final (estado y tiempos).
  - Ejecuta pruebas Node dentro de un contenedor.
  - Construye una imagen Docker del app de ejemplo.
  - Ejecuta un escaneo rápido con Trivy (dentro de Docker).

## 4) Grafana (instancia externa)
1. En tu Grafana, crea un **Data Source** de tipo *Prometheus* que apunte a `http://<prometheus-host>:9090`.
2. Importa el dashboard `grafana/grafana-lab27-dashboard.json`.

## 5) Consultas útiles (en Prometheus)
- `jenkins_job_last_build_result` (0=success, 1=unstable, 2=failure)
- `jenkins_job_last_build_duration_seconds`
- `pipeline_status`
- `pipeline_start_timestamp`, `pipeline_end_timestamp`
- `pipeline_duration_seconds`

## 6) Alertas
- Ya vienen dos reglas de ejemplo: build fallido y pipeline lento.
- Para notificar a Slack/Email, edita `alertmanager/alertmanager.yml` con tu receptor.

## 7) Limpieza
```bash
cd monitoring
docker compose down -v
```

¡Listo! Ejecuta un par de builds de éxito y uno fallido para ver cómo cambian las métricas y alertas.
# lab27
