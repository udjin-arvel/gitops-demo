# Troubleshooting

Краткий разбор типичных проблем стенда **arvelov-gitops-demo**.

## Kind

**`kind create cluster` падает на Docker**

- Docker Desktop / engine запущен.
- Имя кластера свободно: `kind get clusters`.
- Повтор: `kind delete cluster --name arvelov-gitops-demo`, затем снова `kind create ...`.

**`localhost:8080` не открывает podinfo**

- Проброс задан в `kind-config.yaml`: хост `8080` → контейнер `30080`.
- Сервис `podinfo` с `nodePort: 30080`: `kubectl -n podinfo get svc`.
- Порт 8080 на Windows не занят другим процессом (OSPanel, старый port-forward Argo CD).

**Grafana по `localhost` не открывается**

NodePort Grafana — `30081`. В kind проброшен только `30080`. Открывайте через port-forward:

```bash
kubectl -n monitoring port-forward svc/monitoring-grafana 3000:80
```

http://localhost:3000

## Argo CD

**Поды `argocd-*` не становятся Ready**

```bash
kubectl -n argocd get pods
kubectl -n argocd describe pod <name>
kubectl -n argocd logs <name>
```

На слабой машине подождите 1–2 минуты, затем `kubectl -n argocd rollout status deploy/argocd-server`.

**UI Argo CD не на `localhost:8080`**

`8080` занят podinfo. UI:

```bash
kubectl -n argocd port-forward svc/argocd-server 8443:443
```

https://localhost:8443

**Application `Unknown` / `ComparisonError`**

- `repoURL` в `argocd/root-app.yaml` указывает на ваш репозиторий, ветка запушена.
- Репозиторий добавлен в Argo CD (для private — SSH-ключ или token).
- Helm-чарты доступны с машины, где крутится repo-server: `ghcr.io/stefanprodan/charts` (podinfo) и `https://prometheus-community.github.io/helm-charts`.

```bash
kubectl -n argocd get applications
kubectl -n argocd describe application root-app
kubectl -n argocd describe application monitoring
```

**`Sync failed` / namespace не создаётся**

В Application должен быть `syncOptions: CreateNamespace=true`. Иначе создайте namespace вручную:

```bash
kubectl create namespace podinfo
kubectl create namespace monitoring
```

**Корневое приложение не подхватывает `apps/`**

`path: apps` и `directory.recurse: true` в `argocd/root-app.yaml`. Файлы должны быть `Application` (`apiVersion: argoproj.io/v1alpha1`). Сейчас это `apps/podinfo/application.yaml`, `apps/monitoring/crds-application.yaml` и `apps/monitoring/application.yaml`.

**`monitoring` падает, пока нет CRD**

Стек ставится с `skipCrds: true`. CRD ставит отдельное приложение `monitoring-crds` (чарт `prometheus-operator-crds`, ServerSideApply). Сначала дождитесь `Synced` / `Healthy` у `monitoring-crds`, затем смотрите `monitoring`.

На больших CRD client-side apply упирается в лимит 256KiB. В обоих Application мониторинга стоит `ServerSideApply=true`.

## SOPS

**`error loading config` / не тот ключ**

В `.sops.yaml` должен быть **публичный** ключ `age1...`, тот же, которым шифровали `demo-secret.enc.yaml`.

**`failed to get the data key` / `no identity found`**

```bash
export SOPS_AGE_KEY_FILE="$PWD/age-key.txt"
sops -d secrets/demo-secret.enc.yaml
```

Файл `age-key.txt` — приватный ключ, не коммитится.

**В Git попал plaintext**

Коммитьте только `secrets/demo-secret.enc.yaml`. Исходник `demo-secret.yaml` нужен лишь чтобы перегенерировать шифротекст:

```bash
sops -e secrets/demo-secret.yaml > secrets/demo-secret.enc.yaml
```

`demo-secret.enc.yaml` зашифрован ключом из локального `age-key.txt`. Без этого файла (и `SOPS_AGE_KEY_FILE`) `sops -d` не сработает.

**Секрет не появляется в кластере сам**

Argo CD не расшифровывает SOPS. Локально:

```bash
sops -d secrets/demo-secret.enc.yaml | kubectl apply -f -
```

Постоянно — KSOPS или CMP-плагин SOPS. Чарт kube-prometheus-stack этот Secret не использует: пароль Grafana лежит в `monitoring-grafana`.

## Приложения

**`podinfo` CrashLoop / ImagePull**

```bash
kubectl -n podinfo get pods,svc
kubectl -n podinfo describe pod <name>
```

Проверьте, что Docker может тянуть `ghcr.io/stefanprodan/podinfo`. Сервис должен быть NodePort `30080`, страница — http://localhost:8080.

**Prometheus или Grafana Pending**

```bash
kubectl -n monitoring get pods,svc
kubectl -n monitoring describe pod -l app.kubernetes.io/name=grafana
kubectl -n monitoring describe pod -l app.kubernetes.io/name=prometheus
```

Частые причины: CRD ещё не встали (`monitoring-crds`), не хватает памяти (в values у Prometheus request `256Mi`, у Grafana `128Mi`). Persistence в values не включена, PVC не нужен. Alertmanager выключен.

Логин Grafana: `admin`. Пароль:

```bash
kubectl -n monitoring get secret monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 -d; echo
```

## Полезные команды

```bash
kubectl config use-context kind-arvelov-gitops-demo
kubectl -n argocd get applications -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail
kind delete cluster --name arvelov-gitops-demo
```
