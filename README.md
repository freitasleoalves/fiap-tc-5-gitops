# SolidaryTech — GitOps (ArgoCD)

Repositório GitOps dedicado à Fase 5 (Hackathon). Define o estado desejado
de todas as aplicações e addons do cluster Kubernetes da SolidaryTech. O
ArgoCD monitora este repositório e sincroniza automaticamente as mudanças
no AKS provisionado por `fiap-tc-5-terraform`.

Segue exatamente o mesmo padrão (App-of-Apps + ApplicationSet +
Kustomize base/overlays) validado nas Fases 3/4 do ToggleMaster — **a
única mudança estrutural** é que os namespaces dos apps não têm mais
prefixo de produto (`ngo`, `donation`, `volunteer`, `notification`, em vez
de `togglemaster-<app>`), já que este repositório serve só a SolidaryTech.

## Estrutura

```
bootstrap/
  app-of-apps.yaml          # ponto de entrada do ArgoCD (aplicar 1x manualmente)
applicationsets/
  addons.yaml                # Application para os addons do cluster
  apps-appset.yaml            # ApplicationSet: descobre apps/*/overlays/prod
addons/
  kustomization.yaml
  sonarqube/                  # SAST (requisito DevSecOps)
  kube-prometheus-stack/      # Prometheus + Grafana + Alertmanager
  loki/                       # Logs centralizados
  otel-collector-gateway/     # OTel Collector (Deployment) — traces/métricas/logs dos apps
  otel-collector-logs/        # OTel Collector (DaemonSet) — logs de TODO o cluster
  grafana-dashboards/         # Dashboard Overview + Dashboard SRE (SLO/Error Budget)
  velero/                     # Disaster Recovery (Opção A do desafio)
apps/
  ngo/                        # ngo-service (Python/Flask, PostgreSQL)
  donation/                   # donation-service (Go, PostgreSQL + Service Bus — Hot Path)
  volunteer/                  # volunteer-service (Python/Flask, Cosmos DB Table API)
  notification/               # notification-service (consumidor da fila de doações)
```

## Bootstrap (uma vez só)

```bash
az aks get-credentials --resource-group <resource_group_name> --name <aks_cluster_name>
kubectl apply -f bootstrap/app-of-apps.yaml
```

A partir daí o ArgoCD assume o resto sozinho.

## Antes de sincronizar: secrets

Todos os `secrets.yaml` deste repositório (apps + addons) estão com
placeholders (`REPLACE_WITH_...`). Preencha com os valores reais dos
**outputs do Terraform** (`fiap-tc-5-terraform`) antes de deixar o ArgoCD
sincronizar — ou aplique diretamente no cluster via `kubectl create secret
... | kubectl apply -f -` se o GitHub bloquear o push por conter segredo
real (Push Protection — este repo é público, mesma situação já enfrentada
na Fase 4 com a API Key do Datadog).

| Secret | Onde | Vem de (output do Terraform) |
|---|---|---|
| `db-secrets` (ngo) | `apps/ngo/overlays/prod/secrets.yaml` | `postgres_ngo_connection_string` |
| `db-secrets` + `azure-secrets` (donation) | `apps/donation/overlays/prod/secrets.yaml` | `postgres_donation_connection_string`, `servicebus_connection_string` |
| `azure-secrets` (volunteer) | `apps/volunteer/overlays/prod/secrets.yaml` | `cosmosdb_connection_string` |
| `azure-secrets` (notification) | `apps/notification/overlays/prod/secrets.yaml` | `servicebus_connection_string`, `cosmosdb_connection_string` |
| `datadog-api-key` | `addons/otel-collector-gateway/secrets.yaml` | sua API Key do Datadog |
| `cloud-credentials` (Velero) | `addons/velero/secrets.yaml` | `velero_service_principal_client_id/secret`, `velero_tenant_id` |
| `grafana.ini` `domain` | `addons/kube-prometheus-stack/values.yaml` | IP do `ingress-nginx-controller` (só existe após o primeiro `terraform apply`) |
| `addons/velero/values.yaml` (resourceGroup/storageAccount/subscriptionId) | — | `resource_group_name`, `velero_storage_account_name`, `aks_node_resource_group`, `subscription_id` |

## Convenção de nomes dos apps

| Variável | Exemplo | Descrição |
|---|---|---|
| `path.basename` | `donation` | Nome do diretório do app em `apps/` |
| Application name | `app-donation` | Prefixo `app-` + basename |
| Namespace | `donation` | Sem prefixo de produto |

Pra adicionar um novo microsserviço: crie `apps/<novo-app>/` com a
estrutura `base/` + `overlays/prod/` — o ArgoCD descobre e cria a
Application automaticamente via `apps-appset.yaml`.

## Self-Healing: uma peculiaridade desta fase

Cada microsserviço tem seu **próprio repositório** (diferente da Fase 4,
onde o workflow de self-healing morava no repo GitOps). O webhook de
self-healing do Datadog (`fiap-tc-5-terraform/datadog.tf`) dispara
`repository_dispatch` direto no repositório de cada serviço
(`fiap-tc-5-<serviço>`), que precisa ter seu próprio
`.github/workflows/self-heal.yml` (ver Fase C do `CONTEXTO-FASE5.md`).

## Disaster Recovery (Velero)

- `addons/velero/schedule.yaml` define dois backups agendados: um diário
  cobrindo todo o ecossistema, outro horário só do namespace `donation`
  (Hot Path).
- RTO/RPO formalizados no PCN (relatório de entrega) — ver
  `fiap-tc-5-terraform/README.md` para a tabela e o racional.
- Pra restaurar: `velero restore create --from-backup <nome-do-backup>`.
