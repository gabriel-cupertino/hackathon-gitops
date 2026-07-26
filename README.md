# hackathon-gitops — Manifestos Kubernetes (GitOps / ArgoCD)

Repositório **GitOps** da plataforma SolidaryTech (Hackathon Fase 5). Contém os manifestos Kubernetes dos 3 microsserviços, sincronizados automaticamente no cluster **EKS** pelo **ArgoCD**.

> Este repo é a **fonte da verdade** do estado desejado do cluster. Nenhum `kubectl apply` manual — o ArgoCD reconcilia tudo a partir daqui.

## Como funciona

```
Push neste repo → ArgoCD detecta o drift → sincroniza no EKS (self-heal + prune)
```

O CI de [`hackathon-apps`](https://github.com/gabriel-cupertino/hackathon-apps) atualiza automaticamente a **tag da imagem** nos deployments daqui a cada build — fechando o ciclo `código → imagem (ECR) → GitOps → cluster`. As 3 ArgoCD Applications são criadas pelo Terraform em [`hackathon-iac`](https://github.com/gabriel-cupertino/hackathon-iac).

## Estrutura

```
hackathon-gitops/
├── ngo-service/         # deployment, service, hpa, ingress, configmap, job
├── donation-service/    # + servicemonitor (scrape do /metrics pelo Prometheus)
├── volunteer-service/   # deployment, service, hpa, ingress
└── grafana-dashboards/  # dashboard SRE (ConfigMap carregado pelo sidecar do Grafana)
```

Por serviço:

| Manifesto | Função |
|---|---|
| `deployment.yaml` | Pods do serviço — com `requests`/`limits` (rightsizing) e probes |
| `service.yaml` | ClusterIP interno |
| `hpa.yaml` | Horizontal Pod Autoscaler (CPU/memória) |
| `ingress.yaml` | Roteamento via ingress-nginx (`/ngo-service`, `/donation-service`, `/volunteer-service`) |
| `configmap.yaml` | Variáveis de ambiente não sensíveis |
| `job.yaml` | Migração/seed inicial do banco |
| `servicemonitor.yaml` | (donation) expõe o `/metrics` ao Prometheus |

## Convenções importantes

- **`replicas` NÃO é declarado nos deployments** — quem gerencia o número de réplicas é o **HPA**. Declarar `replicas` fixo causaria conflito com o `selfHeal` do ArgoCD (loop de criação/terminação de pods).
- **Rightsizing por uso real:** requests calibrados por observação (`kubectl top` + Grafana) — ex: volunteer 192Mi, ngo 160Mi, donation 128Mi.
- **Dashboard SRE** (`grafana-dashboards/`): painéis de SLO/Error Budget do donation-service, com fallback para não exibir "no-data" quando o cluster está sem tráfego.

## Dashboard SRE

O `sre-donation-service.yaml` é um ConfigMap (label `grafana_dashboard: "1"`) carregado automaticamente pelo sidecar do Grafana. Mostra: SLO de latência, taxa de sucesso, Error Budget consumido/restante, throughput por status code, saúde dos pods.
