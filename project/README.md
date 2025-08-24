# Managed K8s: Online Boutique Platform (Ingress + Monitoring + Logging + CI/CD)

Проектный скелет для защиты: Managed Kubernetes кластер, публичный ingress + домен (nip.io), мониторинг (kube-prometheus-stack) c алертами, централизованные логи (Loki+Promtail), CI/CD (GitLab CI) для пересборки `frontend`.

## 0) Требования и инструменты

- **Kubernetes**: Managed-кластер в любом облаке (YC/EKS/GKE/DO).
- **Утилиты:** `kubectl`, `helm` (v3), `envsubst` (часть пакета `gettext`), `yq` (опц.).
- **Доступ в интернет** для установки чарта и деплоя приложения.
- **Публичный IP** у `ingress-nginx` и домен вида `shop.<EXTERNAL-IP>.nip.io`.

## 1) Кластер (пример для Yandex Cloud, можно пропустить если кластер уже есть)

Смотрите `infra/yc/*` — там базовые скрипты создания и очистки. Либо создайте в веб‑консоли.
Получите контекст доступа и проверьте:
```bash
yc managed-kubernetes cluster get-credentials demo-cluster --external --force
kubectl get nodes
```

## 2) Установка ingress-nginx и получение домена

```bash
./scripts/01_install_ingress.sh
# дождитесь EXTERNAL-IP у ingress-nginx-controller и запомните его
kubectl -n ingress-nginx get svc
```

Хост для магазина будет: `shop.<EXTERNAL-IP>.nip.io` (пример: `shop.203.0.113.10.nip.io`).

## 3) Деплой Online Boutique + Ingress

```bash
./scripts/02_deploy_app.sh
# скрипт сам определит EXTERNAL-IP и сгенерирует Ingress на shop.<IP>.nip.io
kubectl get ingress
```

Проверьте в браузере: `http://shop.<EXTERNAL-IP>.nip.io` — откроется главная страница магазина.

## 4) Мониторинг (Prometheus+Grafana+Alertmanager) + алерты

```bash
./scripts/03_install_monitoring.sh
# Grafana НЕ публикуется наружу — см. README ниже для логина и дашбордов
```

В составе включены:
- включён сбор метрик `ingress-nginx` (ServiceMonitor);
- пример алерта на высокий уровень 5xx;
- преднастройки Grafana (admin/admin123).

Сделайте скриншоты дашбордов/алертов в `docs/` для защиты.

## 5) Централизованные логи (Loki + Promtail)

```bash
./scripts/04_install_logging.sh
```

Откройте Grafana → Explore → Loki и найдите логи по `namespace="default"` и по имени пода `frontend`.

## 6) CI/CD (GitLab CI)

Файл: `.gitlab-ci.yml` — сборка образа фронта и деплой через `kubectl set image`.
В настройках проекта GitLab добавьте **CI Variables**:
- `REGISTRY` — адрес реестра (например, `cr.yandex/<registry-id>` или `registry.gitlab.com/<user>/<repo>`),
- `REG_USER`, `REG_PASSWORD` — учётка реестра,
- `KUBECONFIG_B64` — содержимое kubeconfig, **base64**.
- (опц.) `IMAGE_TAG` — если хотите фиксированный тег, по умолчанию берётся `CI_COMMIT_SHORT_SHA`.

Пайплайн: коммит в ветку → `build` → `push` → на `main` выполняется `deploy` с обновлением тега контейнера в `Deployment/frontend`.

## 7) Полезные команды проверки

```bash
# Внешний IP ingress
kubectl -n ingress-nginx get svc ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}'; echo

# Состояние приложения
kubectl get pods -o wide
kubectl -n default get deploy,svc,ingress

# Мониторинг
kubectl -n observability get pods
kubectl -n observability get prometheusrule

# Логи фронта
kubectl logs deploy/frontend -c server -f --tail=100
```

## 8) Очистка (чтобы не тратить деньги)

```bash
./scripts/90_cleanup.sh
# Удалит платформенные релизы и приложение. Кластер удаляйте самостоятельно (или infra/yc/cleanup.sh).
```

---

### Где что лежит
```
platform/ingress/values.yaml           # ingress-nginx (LB + metrics + servicemonitor)
platform/monitoring/values.yaml        # kube-prometheus-stack overrides
platform/monitoring/alerts/ingress-5xx.yaml # пример алерта на высокий 5xx
platform/logging/values.yaml           # loki-stack (promtail включён)

apps/online-boutique/ingress.yaml.tmpl # шаблон Ingress для фронта
scripts/*.sh                           # инсталляция по шагам
.gitlab-ci.yml                         # CI/CD для фронта
```

**Grafana:** логин/пароль: `admin / admin123` (меняем в `platform/monitoring/values.yaml`).  
**Безопасность:** Grafana не публикуется наружу в этом скелете. Для защиты достаточно скриншотов.

