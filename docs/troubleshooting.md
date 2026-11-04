# Troubleshooting

Краткий разбор типичных проблем стенда **arvelov-gitops-demo**.

## Kind

**`kind create cluster` падает на Docker**

- Docker Desktop / engine запущен.
- Имя кластера свободно: `kind get clusters`.
- Повтор: `kind delete cluster --name arvelov-gitops-demo`, затем снова `kind create ...`.

**`localhost:8080` не отвечает**

- Проброс задан в `kind-config.yaml`: хост `8080` → контейнер `30080`.
- Сервис с `nodePort: 30080` существует: `kubectl get svc -A`.
- Порт 8080 на Windows не занят другим процессом (часто это port-forward Argo CD или OSPanel).

## Argo CD

**Поды `argocd-*` не становятся Ready**

```bash
kubectl -n argocd get pods
kubectl -n argocd describe pod <name>
kubectl -n argocd logs <name>
```

На слабой машине подождите 1–2 минуты, затем `kubectl -n argocd rollout status deploy/argocd-server`.

**Application `Unknown` / `ComparisonError`**

- `repoURL` в манифестах указывает на ваш репозиторий, ветка запушена.
- Репозиторий добавлен в Argo CD (для private — SSH-ключ или token).
- Helm-чарты `podinfo` и `grafana` доступны с машины, где крутится repo-server.

```bash
kubectl -n argocd get applications
kubectl -n argocd describe application root-app
```

**`Sync failed` / namespace не создаётся**

В Application должен быть `syncOptions: CreateNamespace=true`. Иначе создайте namespace вручную:

```bash
kubectl create namespace podinfo
kubectl create namespace monitoring
```

**Корневое приложение не подхватывает `apps/`**

`path: apps` и `directory.recurse: true` в `argocd/root-app.yaml`. Файлы должны быть `Application` (`apiVersion: argoproj.io/v1alpha1`).

## SOPS

**`error loading config` / не тот ключ**

В `.sops.yaml` должен быть **публичный** ключ `age1...`, тот же, которым шифровали `demo-secret.enc.yaml`.

**`failed to get the data key` / `no identity found`**

```bash
export SOPS_AGE_KEY_FILE="$PWD/age-key.txt"
sops -d secrets/demo-secret.enc.yaml
```

Файл `age-key.txt` — приватный ключ, не коммитится.

**Grafanaбор в Git — plaintext**

Коммитьте только `secrets/demo-secret.enc.yaml`. Исходник `demo-secret.yaml` нужен лишь чтобы перегенерировать шифротекст:

```bash
sops -e secrets/demo-secret.yaml > secrets/demo-secret.enc.yaml
```

`demo-secret.enc.yaml` зашифрован ключом из локального `age-key.txt`. Без этого файла (и `SOPS_AGE_KEY_FILE`) `sops -d` не сработает.

**Grafana не стартует: нет `demo-secret`**

Helm Grafana ждёт Secret `demo-secret` в namespace `monitoring` с ключами `admin-user` и `admin-password`. Argo CD сам SOPS не расшифровывает. Варианты:

1. Локально: `sops -d secrets/demo-secret.enc.yaml | kubectl apply -f -`
2. Постоянно: KSOPS / CMP-плагин SOPS в Argo CD

## Приложения

**`podinfo` CrashLoop / ImagePull**

```bash
kubectl -n podinfo get pods,svc
kubectl -n podinfo describe pod <name>
```

Проверьте, что Docker может тянуть `ghcr.io/stefanprodan/podinfo`.

**Grafana Pending**

```bash
kubectl -n monitoring get pods,svc,secret
kubectl -n monitoring describe pod -l app.kubernetes.io/name=grafana
```

Частая причина — отсутствует `demo-secret` (см. выше). Persistence в values выключен, PVC не нужен.

## Полезные команды

```bash
kubectl config use-context kind-arvelov-gitops-demo
kubectl -n argocd get applications -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail
kind delete cluster --name arvelov-gitops-demo
```
