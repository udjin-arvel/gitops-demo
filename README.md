# arvelov-gitops-demo

Локальный GitOps-стенд: **Kind** + **Argo CD** (App of Apps) + **SOPS**.

В кластере поднимаются два приложения:

- `podinfo` — учебный HTTP-сервис
- `monitoring` — Grafana

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

Порт `8080` на хосте проброшен на NodePort `30080` в кластере.

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

UI через NodePort (после патча сервиса) или port-forward:

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

Открыть https://localhost:8080 (логин `admin`).

Чтобы ходить на UI по `http://localhost:8080` без port-forward, переведите `argocd-server` в NodePort `30080`.

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

Расшифрованный секрет в кластер (пока без плагина в Argo CD):

```bash
sops -d secrets/demo-secret.enc.yaml | kubectl apply -f -
```

## 4. App of Apps

1. Запушьте репозиторий.
2. В `argocd/root-app.yaml` и в `apps/*/application.yaml` укажите свой `repoURL`.
3. Зарегистрируйте репозиторий в Argo CD (UI → Settings → Repositories или `argocd repo add`).
4. Примените корневое приложение:

```bash
kubectl apply -f argocd/root-app.yaml
```

Корневое Application создаст `podinfo` и `monitoring`. При `automated.syncPolicy` они сами синхронизируются.

Проверка:

```bash
kubectl -n argocd get applications
kubectl -n podinfo get pods
kubectl -n monitoring get pods
```

`podinfo` слушает NodePort `30080` → http://localhost:8080 (если порт не занят Argo CD).

## Что не коммитить

- `age-key.txt` и любые приватные ключи
- `secrets/demo-secret.yaml` (plaintext)

## Дальше

Типичные сбои и как их разбирать: [docs/troubleshooting.md](docs/troubleshooting.md).
