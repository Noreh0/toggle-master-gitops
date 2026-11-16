# Observabilidade (Fase 4)

Tudo aqui é entregue pelo ArgoCD. A Application `monitoring` (repo da aplicação, `argocd/monitoring.yaml`)
aponta para `monitoring/applications/`, que contém as Applications filhas (charts Helm com os values deste repo).

```
apps (OTLP/HTTP :4318)
   │
   ▼
OTel Collector "gateway" (Deployment) ──► Prometheus (remote write)   métricas dos apps + RED gerado dos spans
                                      ├─► New Relic (OTLP)            traces / APM / Service Map
                                      └─► Loki (OTLP)                 logs recebidos por OTLP

stdout dos pods toggle-app ──► OTel Collector "logs" (DaemonSet, filelog) ──► Loki
Prometheus + Loki ──► Grafana (dashboards e alertas) ──► PagerDuty / Discord / GitHub Actions (auto-reparo)
```

| Application | Chart | Values |
|---|---|---|
| `kube-prometheus-stack` | prometheus-community/kube-prometheus-stack 88.4.0 | `values/kube-prometheus-stack.yaml` |
| `loki` | grafana-community/loki 18.7.6 | `values/loki.yaml` |
| `otel-collector` | open-telemetry/opentelemetry-collector 0.148.0 | `values/otel-collector-gateway.yaml` |
| `otel-collector-logs` | open-telemetry/opentelemetry-collector 0.148.0 | `values/otel-collector-logs.yaml` |

Decisões:
- **Armazenamento efêmero** (emptyDir) em Prometheus e Loki: o cluster não tem o driver EBS CSI e a stack é
  descartável junto com o `terraform destroy`.
- **Alertmanager desligado**: o alerta é do Grafana Alerting.
- **Métricas RED vêm do conector `spanmetrics`** do Collector (`traces_span_metrics_*`): um span SERVER com
  status ERROR é uma resposta 5xx, igual nos 5 serviços, qualquer que seja a versão da semconv de cada SDK.
- **Segredos nunca ficam no Git**: o `scripts/bootstrap.sh` (repo da aplicação) cria os Secrets
  `newrelic-license`, `grafana-admin` e `grafana-alerting-secrets` no namespace `monitoring`.

As Applications apontam para a branch **main** deste repo: faça o merge antes de subir o cluster.
