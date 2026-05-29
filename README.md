# Kubernetes Cluster Config

GitOps-конфигурация Kubernetes-кластера для базовой инфраструктуры homelab/dev-окружения.

Репозиторий рассчитан на управление через Argo CD: сначала вручную применяются bootstrap-манифесты, затем Argo CD синхронизирует инфраструктурные приложения из каталога `infrastructure`.

## Что разворачивается

| Компонент | Назначение | Версия/источник |
| --- | --- | --- |
| MetalLB | LoadBalancer для bare-metal/kind-кластера | `metallb` chart `v0.16.1` |
| Envoy Gateway | Gateway API ingress layer | `eg` chart `v1.7.2` |
| cert-manager | Выпуск TLS-сертификатов | `cert-manager` chart `v1.20.2` |
| trust-manager | Распространение CA bundle | `trust-manager` chart `v0.22.1` |
| Argo CD | GitOps control plane | `argo-cd` chart `v2.2.5` |

## Структура

```text
.
├── bootstrap
│   ├── cluster-bootstrap.yaml
│   └── infrastucture-project.yaml
└── infrastructure
    ├── argocd
    │   ├── application.yaml
    │   └── values.yaml
    ├── certmanager
    │   ├── application.yaml
    │   └── manifests
    │       └── ca.yaml
    ├── gateway
    │   ├── application.yaml
    │   └── manifests
    │       ├── gateway.yaml
    │       └── gatewayclass.yaml
    └── metallb
        ├── application.yaml
        └── manifests
            ├── ippaddresspool.yaml
            └── l2advertisment.yaml
```

## Как это работает

1. `bootstrap/infrastucture-project.yaml` создаёт Argo CD project `infrastructure`.
2. `bootstrap/cluster-bootstrap.yaml` создаёт root application `cluster-bootstrap`.
3. Root application рекурсивно ищет `*/application.yaml` в каталоге `infrastructure`.
4. Каждое infrastructure-приложение устанавливает Helm chart и, где нужно, дополнительные YAML-манифесты из этого репозитория.

Порядок синхронизации задаётся annotation `argocd.argoproj.io/sync-wave`:

| Wave | Application |
| --- | --- |
| `1` | `metallb` |
| `2` | `envoy-gateway` |
| `3` | `cert-manager` |
| `4` | `argocd` |

## Bootstrap

Предварительные условия:

- в кластере уже установлен Argo CD;
- есть доступ к кластеру через `kubectl`;
- namespace `argocd` существует или будет создан заранее;
- репозиторий `https://github.com/vladoz77/k8s-cluster-config.git` доступен из Argo CD.

Применить bootstrap:

```bash
kubectl apply -f bootstrap/infrastucture-project.yaml
kubectl apply -f bootstrap/cluster-bootstrap.yaml
```

Проверить синхронизацию:

```bash
kubectl get applications -n argocd
kubectl get pods -n metallb-system
kubectl get pods -n envoy-gateway-system
kubectl get pods -n cert-manager
```

## Доступ к Argo CD

Argo CD настроен на домен:

```text
argocd.dev.local
```

HTTPRoute создаётся через Helm values Argo CD и подключается к Gateway:

```text
envoy-gateway-system/envoy-gateway
```

Для локального доступа нужно направить `argocd.dev.local` на внешний IP Envoy Gateway. В текущей конфигурации MetalLB выдаёт адреса из пула:

```text
172.18.255.200-172.18.255.250
```

Пример проверки IP:

```bash
kubectl get svc -n envoy-gateway-system
```

После этого добавьте запись в локальный DNS или `/etc/hosts`.

## TLS и CA

`cert-manager` создаёт:

- `ClusterIssuer/selfsigned-issuer`;
- CA certificate `cert-manager/ca`;
- `ClusterIssuer/ca-issuer`;
- trust-manager `Bundle/trust-ca`.

Gateway `envoy-gateway` использует issuer `ca-issuer` и TLS secret `envoy-tls-secret` для `*.dev.local`.

## Что менять под свой кластер

- `infrastructure/metallb/manifests/ippaddresspool.yaml` - диапазон IP-адресов MetalLB.
- `infrastructure/gateway/manifests/gateway.yaml` - hostname `*.dev.local`, listeners и TLS secret.
- `infrastructure/argocd/values.yaml` - домен Argo CD, параметры сервера и admin password hash.
- `bootstrap/*.yaml` и `infrastructure/*/application.yaml` - `repoURL`, если репозиторий переехал.

## Важные замечания

- В `infrastructure/argocd/values.yaml` хранится hash admin-пароля Argo CD. Не коммитьте реальные секреты в публичный репозиторий.
- `server.extraArgs: ["--insecure"]` отключает TLS на самом Argo CD server, потому что TLS завершается на Envoy Gateway.
- Включён automated sync для root application и infrastructure applications. Argo CD будет применять изменения из Git автоматически.
- В root application включены `prune` и `selfHeal`, поэтому удалённые из Git ресурсы могут быть удалены из кластера.

## Полезные команды

```bash
# Статус Argo CD applications
kubectl get applications -n argocd

# Детали конкретного приложения
kubectl describe application metallb -n argocd

# Gateway и listeners
kubectl get gateway -n envoy-gateway-system

# MetalLB IP pools
kubectl get ipaddresspool -n metallb-system

# cert-manager issuers
kubectl get clusterissuer
```
