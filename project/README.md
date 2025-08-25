# Managed K8s: Online Boutique (Ingress, Monitoring, Logging, CI/CD)

Managed Kubernetes‑кластер, публичный ingress + домен (`nip.io`), мониторинг (kube‑prometheus‑stack) c алертами,  логи (Loki+Promtail), CI/CD (GitHub Actions) для сборки и раскатки `frontend` из Online Boutique.

---

## Содержание
- [Требования](#требования)
- [Быстрый старт](#быстрый-старт)
- [Ingress и домен](#ingress-и-домен)
- [Деплой приложения](#деплой-приложения)
- [Мониторинг (Prometheus/Grafana)](#мониторинг-prometheusgrafana)
- [Логи (Loki/Promtail)](#логи-lokipromtail)
- [CI/CD (GitHub Actions + GHCR)](#cicd-github-actions--ghcr)
- [Проверки](#проверки)
- [Очистка](#очистка)
- [Структура репозитория](#структура-репозитория)
- [Требуемые скриншоты для отчёта](#требуемые-скриншоты-для-отчёта)
- [FAQ](#faq)

---

## Требования

- **Kubernetes**: Managed‑кластер в облаке (YC).
- **Утилиты на рабочей машине:** `kubectl`, `helm` (v3), `envsubst` (`gettext`), `yq` (по желанию).
- **Публичный IP** у `ingress-nginx` и домен вида `shop.<EXTERNAL-IP>.nip.io`.

> Пример ниже использует неймспейс `observability` для мониторинга/логов и `default` для приложения.

---

## Быстрый старт

```bash
# 1) Ingress
./scripts/01_install_ingress.sh
kubectl -n ingress-nginx get svc ingress-nginx-controller

# 2) Приложение Online Boutique + Ingress
./scripts/02_deploy_app.sh
kubectl get ingress

# 3) Мониторинг (Prometheus/Grafana/Alertmanager)
./scripts/03_install_monitoring.sh
kubectl -n observability get pods

# 4) Логи (Loki+Promtail)
./scripts/04_install_logging.sh
kubectl -n observability get pods -l app.kubernetes.io/name=promtail

# 5) Локальный доступ (порт‑форварды)
kubectl -n observability port-forward svc/kps-kube-prometheus-stack-prometheus 9090:9090
kubectl -n observability port-forward deploy/kps-grafana 3000:3000
```

В браузере:
- **Prometheus:** http://localhost:9090/targets (все цели зелёные)
- **Grafana:** http://localhost:3000  (admin/admin123)

---

## Ingress и домен

Скрипт `01_install_ingress.sh` ставит `ingress-nginx` с включёнными метриками и ServiceMonitor.
После установки получи внешний IP балансировщика:

```bash
kubectl -n ingress-nginx get svc ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}'; echo
```

Хост магазина будет `shop.<EXTERNAL-IP>.nip.io` (пример: `shop.203.0.113.10.nip.io`).

---

## Деплой приложения

```bash
./scripts/02_deploy_app.sh
# скрипт сам определит IP и сгенерирует Ingress на shop.<IP>.nip.io
kubectl get ingress
```

`http://shop.<EXTERNAL-IP>.nip.io`.

---

## Мониторинг (Prometheus/Grafana)

```bash
./scripts/03_install_monitoring.sh
kubectl -n observability get pods
```

- Установлен `kube-prometheus-stack`.
- Включён сбор метрик `ingress-nginx` (ServiceMonitor).
- Пример **алерта** на высокий уровень 5xx: `platform/monitoring/alerts/ingress-5xx.yaml`.
- Логин в Grafana: см. секрет (или значение из values):
  ```bash
  kubectl -n observability get secret kps-grafana -o jsonpath="{.data.admin-password}" | base64 -d; echo
  ```

### Полезные дашборды
- K8s / Nodes / Pods / Workloads (идут в составе чарта).
- Свой дашборд для ingress‑логов можно добавить ConfigMap’ом (пример ниже в разделе **Логи**).

---

## Логи (Loki/Promtail)

```bash
./scripts/04_install_logging.sh
kubectl -n observability get pods -l app.kubernetes.io/name=promtail
```

В Grafana → **Explore** → источник **Loki**.
Примеры запросов **LogQL**:

```logql
{namespace="default", container="server"}
```

Ошибки 5xx по ingress‑контроллеру (если включён JSON‑парсинг в promtail):
```logql
sum by (status) (
  count_over_time({namespace="ingress-nginx", container="controller"} | json | status >= 500 [5m])
)
```

### Примечание для containerd
Если нода без `/var/lib/docker/containers`, promtail может падать. Включённые монтирования: `/var/log/pods` и `/var/log/containers`. Если нужно, уберите docker‑путь в `platform/logging/values.yaml` и обновите:
```bash
helm -n observability upgrade --install loki grafana/loki-stack -f platform/logging/values.yaml
```

### Свой дашборд в Grafana
```bash
kubectl -n observability create configmap grafana-dashboard-ingress \
  --from-file=project/platform/monitoring/grafana/ingress-observability.json \
  -o yaml --dry-run=client | kubectl apply -f -

kubectl -n observability label cm grafana-dashboard-ingress grafana_dashboard=1 --overwrite
kubectl -n observability rollout restart deploy/kps-grafana
```

---

## CI/CD (GitHub Actions + GHCR)

Ветка `main` → **build** → **deploy**.

### Секреты репозитория (Settings → Secrets and variables → Actions → Secrets)

- `KUBECONFIG_B64` — kubeconfig в base64 (можно сгенерировать скриптом `infra/k8s/ci/kubeconfig-from-sa.sh`).
- `GHCR_USER` — логин в GitHub (например, `dvoryankin`).
- `GHCR_PAT` — персональный токен с правами `write:packages` (см. «FAQ»).
  > Если репозиторий в организации, которая **не** даёт GitHub Actions публиковать пакеты, публикуйте образ в пространстве пользователя: `ghcr.io/<ваш-логин>/dvoryankin_repo/frontend`.

### Что делает pipeline
1. **Build**: собирает образ `project/src/frontend/Dockerfile` от `gcr.io/google-samples/...:v0.8.0`, пушит в GHCR:
    - `ghcr.io/<owner>/dvoryankin_repo/frontend:<full-sha>`
    - `ghcr.io/<owner>/dvoryankin_repo/frontend:latest`
      Экспортируются `image` и `sha` как outputs.
2. **Deploy** (только для `main`):
    - пишет kubeconfig из `KUBECONFIG_B64`,
    - выполняет:
      ```bash
      kubectl -n default set image deploy/frontend server="${IMAGE}:${SHA}" --record
      kubectl -n default rollout status deploy/frontend --timeout=180s
      ```

### imagePullSecret для приватного GHCR
```bash
kubectl -n default create secret docker-registry ghcr-creds \
  --docker-server=ghcr.io \
  --docker-username=$GHCR_USER \
  --docker-password=$GHCR_PAT \
  --docker-email=you@example.com

kubectl -n default patch sa default -p '{"imagePullSecrets":[{"name":"ghcr-creds"}]}'
```

---

## Проверки

```bash
# Внешний IP ingress
kubectl -n ingress-nginx get svc ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}'; echo

# Приложение
kubectl -n default get deploy,svc,ingress
kubectl -n default describe deploy/frontend | grep -i image:
kubectl -n default logs deploy/frontend -c server --tail=100

# Prometheus
kubectl -n observability get pods -l "app.kubernetes.io/name=grafana"
kubectl -n observability port-forward svc/kps-kube-prometheus-stack-prometheus 9090:9090

# Loki/Promtail
kubectl -n observability get pods -l "app.kubernetes.io/name=promtail" -o wide
kubectl -n observability logs ds/loki-promtail -c promtail --tail=50
```

---

## Очистка

```bash
./scripts/90_cleanup.sh
# удалит платформенные релизы и приложение (кластер удаляется отдельно)
```

---

## Структура репозитория

```
platform/ingress/values.yaml               # ingress-nginx (LB + metrics + servicemonitor)
platform/monitoring/values.yaml            # kube-prometheus-stack overrides
platform/monitoring/alerts/ingress-5xx.yaml
platform/logging/values.yaml               # loki-stack (promtail включён)

apps/online-boutique/ingress.yaml.tmpl     # шаблон Ingress для фронта
scripts/*.sh                               # пошаговые инсталляторы
.github/workflows/cicd.yml                 # GitHub Actions (build+deploy фронта)
```

---

## Скриншоты

1. **GitHub Actions** — зелёные `build` и `deploy`.
2. **GHCR** — страница пакета `dvoryankin_repo/frontend` с последним тегом.
3. **Prometheus** — `/targets` со всеми зелёными целями.
4. **Grafana → Explore (Loki)** — логи контейнера `server` (`namespace="default"`).
5. **Grafana → Дашборд ingress** (если подключали ConfigMap с JSON).
6. **kubectl get ingress** — домен `shop.<IP>.nip.io`.
7. **kubectl describe deploy/frontend | image** — факт обновления образа.

Скрины в `project/docs/`

---


