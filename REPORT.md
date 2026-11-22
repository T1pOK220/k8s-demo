# Kubernetes demo(k8s)


## 1. Локальний запуск і перевірка ендпоінтів

Кроки: клонування форку, `npm install`, `npm run dev`.

Результати перевірки ендпоінтів:

| Ендпоінт | Відповідь |
| --- | --- |
| `/api/status` | `{"status":"ok","appMode":"dev","version":"1.0.0","timestamp":"2026-11-19T17:35:08.734Z"}` |
| `/api/items?filter=a` | `[{"id":1,"name":"Item A"}]` |
| `/api/slow` | `{"status":"ok","delayMs":2000}` |
| `/api/error` | `{"status":"ok","message":"No error"}` |
| `/healthz` | `ok` |
| `/ready` | `{"status":"ready"}` |

---

## 2. Docker

**Збірка образу:**

```bash
docker build -t k8s-demo-app:1.0.0 .
```

**Запуск контейнера:**

```bash
docker run --rm -p 8080:8080 -e SECRET_TOKEN=local k8s-demo-app:1.0.0
```

**Приклад відповіді `/api/status`:**

```json
{"status":"ok","appMode":"dev","version":"1.0.0","timestamp":"2026-11-19T17:46:24.353Z"}
```

---

## 3. Minikube і базовий деплой

- Кластер запущено через `minikube start`; контекст `kubectl` — `minikube`.
- Статус ноди: **Ready**.
- Застосовано маніфести: namespace, configmap, secret, deployment, service.

**Вивід `kubectl get deployment,rs,pods,svc -n k8s-demo`:**

```text
PS C:\Users\Admin\Desktop\k8s-demo\k8s> kubectl get deployment,rs,pods,svc -n k8s-demo
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/k8s-demo-api   2/2     2            2           10m

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/k8s-demo-api-5c45559886   0         0         0       10m
replicaset.apps/k8s-demo-api-65596c66d    2         2         2       2m21s

NAME                               READY   STATUS    RESTARTS   AGE
pod/k8s-demo-api-65596c66d-r58cs   1/1     Running   0          2m15s
pod/k8s-demo-api-65596c66d-vhls5   1/1     Running   0          2m21s

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/k8s-demo-api   ClusterIP   10.103.188.58   <none>        80/TCP    10m
```

**Port-forward** `svc/k8s-demo-api` на `8080:80` і перевірка:

```bash
curl http://localhost:8080/api/items
```

```json
[{"id":1,"name":"Item A"},{"id":2,"name":"Item B"},{"id":3,"name":"Item C"}]
```

---

## 4. Ingress і діагностика

Після налаштування Ingress Controller створено ресурс `k8s-demo-ingress` з host **`k8s-demo.local`**.

### Проблема

Під час перевірки прямого доступу до Minikube за адресою `192.168.49.2` Windows не зміг встановити TCP-з'єднання з портами `80` та `31149`.

### Діагностика

Використані команди: `minikube status`, `kubectl get svc`, `kubectl get endpoints`, `Test-NetConnection`.

Результат:

- Minikube працює через Docker Driver;
- Ingress Controller у стані `Running`;
- Service Ingress Controller має тип `NodePort`.

### Обхід

Використано `kubectl port-forward` напряму до Ingress Controller:

```bash
kubectl port-forward -n ingress-nginx deployment/ingress-nginx-controller 8080:80
```

Після створення тунелю виконано HTTP-запити з заголовком `Host: k8s-demo.local`:

| Запит | Результат |
| --- | --- |
| `GET /api/status` | HTTP 200 OK, JSON із підтвердженням працездатності |
| `GET /api/items` | HTTP 200 OK, список із трьох елементів |

### Висновок

Підтверджено коректну роботу Ingress Controller, маршрутизації за Host, взаємодії із Service та доступності Express API. Проблема стосувалася мережевого доступу до Minikube з Windows, а не конфігурації застосунку.

### Debug-флоу

| Рівень | Команда | Що показує |
| --- | --- | --- |
| Pod | `kubectl get pods -n k8s-demo` / `kubectl logs <pod> -n k8s-demo` | Статус pod-ів, рестарти, логи застосунку |
| Service | `kubectl get svc,endpoints -n k8s-demo` | Тип і порти Service, чи є endpoints (IP pod-ів) за селектором |
| Ingress | `kubectl describe ingress -n k8s-demo` | Host, path, backend-сервіс, події контролера |

Додатково при помилках: `kubectl logs -n ingress-nginx <controller-pod>`, `kubectl get events -n k8s-demo`.

---

## 5. Шлях запиту через `k8s-demo.local`

```text
Клієнт (curl, Host: k8s-demo.local)
        │
        ▼
Ingress Controller (ingress-nginx) — вибирає правило за host/path з k8s-demo-ingress
        │
        ▼
Service k8s-demo-api (ClusterIP, порт 80) — балансує між pod-ами за endpoints
        │
        ▼
Pod k8s-demo-api (контейнер, порт 8080) — Express API
```

Запит із заголовком `Host: k8s-demo.local` потрапляє на Ingress Controller, який за правилами `k8s-demo-ingress` перенаправляє його на Service `k8s-demo-api` (порт 80). Service вибирає один зі справних pod-ів і передає запит на порт 8080 контейнера.

---

## 6. Власний сервіс

### Посилання на репозиторій

GitHub: <https://github.com/T1pOK220/parfumes-store>

### Короткий опис

**Perfumes Store API** — серверна частина вебзастосунку для магазину парфумерії на Node.js та Express. Сервіс забезпечує обмін даними між frontend і базою даних PostgreSQL, обробляє HTTP-запити та відповідає за бізнес-логіку.

Основні групи ендпоінтів:

- `/api/perfumes` — каталог парфумів;
- `/api/users` — користувачі;
- `/api/cart` — кошик;
- `/api/orders` — замовлення;
- `/api/favourites` — обрані товари.

Для перевірки доступності використовується ендпоінт `/health`.

### Dockerfile


```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
EXPOSE 8080
CMD ["npm","run","start"]

```

### Збірка та запуск

```bash
docker build -t parfumes-api-node:1.0.0 .
docker run --rm -p 8081:8080 --env-file .env parfumes-api-node:1.0.0
```

### Приклад відповіді

```bash
curl http://localhost:8081/health
```

```text
{"status":"ok"}
```