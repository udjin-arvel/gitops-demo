# arvelov-gitops-demo

Локальный GitOps-стенд: **Kind** + **Argo CD** (App of Apps) + **SOPS**.

Корневое Application `root-app` подхватывает каталог `apps/` и создаёт дочерние приложения:

- `podinfo` — учебный HTTP-сервис
- `monitoring-crds` — CRD Prometheus Operator
- `monitoring` — kube-prometheus-stack (Prometheus + Grafana)

Секреты в Git хранятся только в зашифрованном виде (`secrets/demo-secret.enc.yaml`).

## Структура

```
.
├── README.md
├── kind-config.yaml
├── argocd/
│   └── root-app.yaml
├── apps/
│   ├── podinfo/
│   │   └── application.yaml
│   └── monitoring/
│       ├── crds-application.yaml
│       └── application.yaml
├── secrets/
│   ├── demo-secret.yaml          # plaintext, only for local generation
│   └── demo-secret.enc.yaml      # encrypted, committed to Git
├── .sops.yaml
└── docs/
    └── troubleshooting.md
```

## Требования

Инструменты ставим и запускаем в **WSL2** (каталог проекта: `/mnt/c/OSPanel/home/k8s-example`): Docker, kind, kubectl, helm, sops, age. Argo CD читает манифесты из Git.

## 1. Кластер Kind

```bash
kind create cluster --name arvelov-gitops-demo --config kind-config.yaml
kubectl cluster-info --context kind-arvelov-gitops-demo
```

В `kind-config.yaml` проброшен один порт: хост `8080` → NodePort `30080`. На нём слушает `podinfo`.

## 2. Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server
```

Начальный пароль admin:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

UI только через port-forward. Порт `8080` на хосте уже занят `podinfo`:

```bash
kubectl -n argocd port-forward svc/argocd-server 8443:443
```

Открыть https://localhost:8443 (логин `admin`). В списке Applications будет `root-app` и дочерние приложения.

## 3. SOPS

Ключ age уже лежит в `age-key.txt` (в Git не коммитить). Публичный ключ прописан в `.sops.yaml`.

```bash
export SOPS_AGE_KEY_FILE="$PWD/age-key.txt"
```

Перешифровать секрет после правок plaintext:

```bash
sops -e secrets/demo-secret.yaml > secrets/demo-secret.enc.yaml
```

В Git уходит только `.enc.yaml`.

`secrets/demo-secret.yaml` оставьте локально или в `.gitignore`.

Расшифровка для проверки:

```bash
sops -d secrets/demo-secret.enc.yaml
```

Расшифрованный секрет в кластер (Argo CD сам SOPS не расшифровывает):

```bash
sops -d secrets/demo-secret.enc.yaml | kubectl apply -f -
```

## 4. App of Apps

1. Запушьте репозиторий.
2. В `argocd/root-app.yaml` укажите свой `repoURL`, если он отличается.
3. Зарегистрируйте репозиторий в Argo CD (UI → Settings → Repositories или `argocd repo add`). Helm-репозитории podinfo и Prometheus отдельно регистрировать не нужно: URL чартов уже в Application.
4. Примените корневое приложение:

```bash
kubectl apply -f argocd/root-app.yaml
```

`root-app` создаст `podinfo`, `monitoring-crds` и `monitoring`. При `automated` они синхронизируются сами. Сначала должны встать CRD (`monitoring-crds`), затем стек.

Проверка:

```bash
kubectl -n argocd get applications
kubectl -n podinfo get pods,svc
kubectl -n monitoring get pods,svc
```

## Куда заходить

| Что | Как открыть |
| --- | --- |
| `podinfo` | http://localhost:8080 |
| Argo CD и `root-app` | `kubectl -n argocd port-forward svc/argocd-server 8443:443`, затем https://localhost:8443 |
| Grafana | `kubectl -n monitoring port-forward svc/monitoring-grafana 3000:80`, затем http://localhost:3000 |

`root-app` — это Application в Argo CD, отдельного HTTP-сервиса у него нет. Его статус виден в UI или так:

```bash
kubectl -n argocd get application root-app
kubectl -n argocd describe application root-app
```

Пароль Grafana берётся из секрета чарта (логин по умолчанию `admin`):

```bash
kubectl -n monitoring get secret monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 -d; echo
```

NodePort Grafana — `30081`, но в `kind-config.yaml` он на хост не проброшен, поэтому с Windows на него напрямую не зайти.

## Что не коммитить

- `age-key.txt` и любые приватные ключи
- `secrets/demo-secret.yaml` (plaintext)

## Дальше

Типичные сбои и как их разбирать: [docs/troubleshooting.md](docs/troubleshooting.md).
