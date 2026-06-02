# Kubernetes Cluster Config

GitOps-конфигурация Kubernetes-кластера для базовой инфраструктуры homelab/dev-окружения.

Репозиторий рассчитан на управление через Argo CD: сначала вручную применяются bootstrap-манифесты, затем root application `cluster-bootstrap` синхронизирует инфраструктурные приложения из каталога `infrastructure`.

## Что разворачивается

| Компонент | Назначение | Источник |
| --- | --- | --- |
| MetalLB | LoadBalancer для bare-metal/kind-кластера | Helm chart `metallb`, `v0.16.1` |
| Envoy Gateway CRDs | CRD Envoy Gateway и Gateway API | Helm chart `gateway-crds-helm`, `v1.8.0` |
| Envoy Gateway | Gateway API ingress layer | Helm chart `gateway-helm`, `v1.8.0` |
| cert-manager | Выпуск TLS-сертификатов | Helm chart `cert-manager`, `v1.20.2` |
| trust-manager | Распространение CA bundle | Helm chart `trust-manager`, `0.22.1` |
| Argo CD | GitOps control plane | Helm chart `argo-cd`, `9.5.17` |

Дополнительные YAML-манифесты из репозитория создают MetalLB IP pool, GatewayClass/Gateway, CA issuer и trust-manager Bundle.

## Структура

```text
.
├── bootstrap
│   ├── cluster-bootstrap.yaml
│   └── infrastucture-project.yaml
└── infrastructure
    ├── argocd
    │   ├── argocd-application.yaml
    │   └── values.yaml
    ├── certmanager
    │   ├── certmanager-application.yaml
    │   ├── certmanager-manifest-application.yaml
    │   └── manifests
    │       └── ca.yaml
    ├── gateway
    │   ├── gateway-application.yaml
    │   ├── gateway-crd-application.yaml
    │   ├── gateway-manifest-application.yaml
    │   └── manifests
    │       ├── gateway.yaml
    │       └── gatewayclass.yaml
    ├── metallb
    │   ├── metallb-application.yaml
    │   ├── metallb-manifest-application.yaml
    │   └── manifests
    │       ├── ippaddresspool.yaml
    │       └── l2advertisment.yaml
    └── trustmanager
        ├── trustmanager-application.yaml
        ├── trustmanager-manifest-application.yaml
        └── manifests
            └── ca-bundle.yaml
```

## Как это работает

1. `bootstrap/infrastucture-project.yaml` создаёт Argo CD project `infrastructure`.
2. `bootstrap/cluster-bootstrap.yaml` создаёт root application `cluster-bootstrap`.
3. Root application читает каталог `infrastructure` рекурсивно и подхватывает файлы по маске `*/*-application.yaml`.
4. Infrastructure applications устанавливают Helm chart'ы и применяют дополнительные манифесты из этого же репозитория.

Порядок синхронизации задаётся annotation `argocd.argoproj.io/sync-wave`:

| Wave | Application | Что делает |
| --- | --- | --- |
| `1` | `metallb` | Устанавливает MetalLB |
| `1` | `envoy-gateway-crds` | Устанавливает CRD Envoy Gateway/Gateway API |
| `2` | `metallb-manifest` | Создаёт `IPAddressPool` и `L2Advertisement` |
| `3` | `envoy-gateway` | Устанавливает Envoy Gateway |
| `3` | `envoy-gateway-manifest` | Создаёт `GatewayClass` и `Gateway` |
| `4` | `cert-manager` | Устанавливает cert-manager с CRD и Gateway API support |
| `5` | `trust-manager` | Устанавливает trust-manager |
| `6` | `cert-manager-manifest` | Создаёт self-signed issuer, CA certificate и `ca-issuer` |
| `7` | `trust-manager-manifest` | Создаёт `Bundle/trust-ca` |
| `8` | `argocd` | Обновляет Argo CD через Helm chart |

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

Проверить, что Argo CD увидел приложения:

```bash
kubectl get applications -n argocd
kubectl describe application cluster-bootstrap -n argocd
```

Проверить базовые namespace и workloads:

```bash
kubectl get pods -n metallb-system
kubectl get pods -n envoy-gateway-system
kubectl get pods -n cert-manager
```

## Сеть и Gateway

MetalLB выдаёт адреса из пула:

```text
172.18.255.200-172.18.255.250
```

Envoy Gateway создаётся в namespace `envoy-gateway-system`:

- `GatewayClass/envoy-gateway-class`;
- `Gateway/envoy-gateway`;
- listener `http` на порту `80`;
- listener `https` на порту `443` для `*.dev.local`.

Gateway принимает маршруты из любых namespace:

```yaml
allowedRoutes:
  namespaces:
    from: All
```

Проверить Gateway и внешний IP:

```bash
kubectl get gateway -n envoy-gateway-system
kubectl get svc -n envoy-gateway-system
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

Для локального доступа нужно направить `argocd.dev.local` на внешний IP Envoy Gateway. Добавьте запись в локальный DNS или `/etc/hosts`.

## TLS и CA

`cert-manager` создаёт:

- `ClusterIssuer/selfsigned-issuer`;
- CA certificate `cert-manager/ca`;
- secret `cert-manager/ca-secret`;
- `ClusterIssuer/ca-issuer`.

`Gateway/envoy-gateway` использует issuer `ca-issuer` и TLS secret `envoy-tls-secret` для `*.dev.local`.

`trust-manager` создаёт `Bundle/trust-ca`, который читает CA certificate из `ca-secret` и публикует bundle в ConfigMap `trust-ca` с ключом `trust-bundle.pem`. Argo CD repo-server монтирует этот bundle как `/etc/ssl/certs/ca.pem`, чтобы доверять локальному CA.

Проверить TLS-ресурсы:

```bash
kubectl get clusterissuer
kubectl get certificate -n cert-manager
kubectl get bundle -n cert-manager
kubectl get configmap trust-ca -n cert-manager
```

## Что менять под свой кластер

- `infrastructure/metallb/manifests/ippaddresspool.yaml` - диапазон IP-адресов MetalLB.
- `infrastructure/gateway/manifests/gateway.yaml` - hostname `*.dev.local`, listeners, TLS secret и issuer.
- `infrastructure/argocd/values.yaml` - домен Argo CD, HTTPRoute, параметры сервера и admin password hash.
- `bootstrap/*.yaml` и `infrastructure/*/*-application.yaml` - `repoURL`, если репозиторий переехал.
- `infrastructure/certmanager/manifests/ca.yaml` - subject CA certificate, если нужен другой локальный CA.

## Важные замечания

- В `infrastructure/argocd/values.yaml` хранится hash admin-пароля Argo CD. Не коммитьте реальные секреты в публичный репозиторий.
- `server.extraArgs: ["--insecure"]` отключает TLS на самом Argo CD server, потому что TLS завершается на Envoy Gateway.
- Включён automated sync для root application и infrastructure applications. Argo CD будет применять изменения из Git автоматически.
- У root application включены `prune` и `selfHeal`, поэтому удалённые из Git ресурсы могут быть удалены из кластера.
- У большинства infrastructure applications включён `CreateNamespace=true`; namespace создаются Argo CD при синхронизации.

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

# trust-manager bundles
kubectl get bundle -n cert-manager
```
