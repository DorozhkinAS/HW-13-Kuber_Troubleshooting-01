# Домашнее задание Troubleshooting" - `Дорожкин Артем`

### Задача 1
  

#### Этап 1: Выявлены две проблемы

1.  **Ошибка загрузки образа (`ErrImagePull`):**
    *   Исходный образ `radial/busyboxplus:curl` устарел и имеет неподдерживаемый формат манифеста (`schema v1`).
    *   Современные системы (containerd) не могут его загрузить, из-за чего поды не запускаются.

2.  **Ошибка DNS (`Could not resolve host`):**
    *   Приложение использует короткое имя сервиса `auth-db` в команде `curl`.
    *   Сервис `auth-db` находится в другом неймспейсе (`data`), а не в `web`.
    *   Короткое имя резолвится только в пределах своего неймспейса, поэтому сервис не находится.
.
.

<img width="1062" height="287" alt="image" src="https://github.com/user-attachments/assets/334cc785-e381-4682-9da3-eefd3834da05" />

.
.
#### Этап 2: Частичное исправление (промежуточный тест)

1.  **Замена образа:**
    *   Устаревший образ `radial/busyboxplus:curl` заменен на рабочий `curlimages/curl:latest`.
    *   Поды успешно запустились и перешли в статус `Running`.

2.  **Результат промежуточного теста:**
    *   Ошибка `ErrImagePull` исчезла, но осталась ошибка **DNS** (`Could not resolve host: auth-db`).
    *   Это доказало, что проблема с образом решена, но проблема с DNS остаётся.
.
.

<img width="420" height="317" alt="image" src="https://github.com/user-attachments/assets/4a59e94f-ec85-45ed-aa60-10506cb7b210" />

.
.
#### Этап 3: Окончательное исправление (финальный манифест)

В финальном манифесте (`task-fixed.yaml`) были внесены следующие изменения:

1.  **Образ:** Оставлен рабочий образ `curlimages/curl:latest`.
2.  **DNS-имя:** В команде `curl` короткое имя `auth-db` заменено на **полное DNS-имя** `auth-db.data.svc.cluster.local`.
.
.
task-fixed.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-consumer
  namespace: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-consumer
  template:
    metadata:
      labels:
        app: web-consumer
    spec:
      containers:
      - command:
        - sh
        - -c
        - while true; do curl auth-db.data.svc.cluster.local; sleep 5; done
        image: curlimages/curl:latest
        name: busybox
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-db
  namespace: data
spec:
  replicas: 1
  selector:
    matchLabels:
      app: auth-db
  template:
    metadata:
      labels:
        app: auth-db
    spec:
      containers:
      - image: nginx:1.19.1
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: auth-db
  namespace: data
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: auth-db

```

<img width="702" height="409" alt="image" src="https://github.com/user-attachments/assets/547b7cd8-2718-4b74-bbb3-036cf9208c89" />

<img width="538" height="189" alt="image" src="https://github.com/user-attachments/assets/df501be0-b253-4f1e-9f0f-e9423fb63743" />

.
.
**Результат проверки:**
*   Поды `web-consumer` находятся в статусе `Running`.
*   В логах `auth-db` видны успешные входящие запросы (HTTP 200).
*   Проблема полностью решена.

## 📝 Итоговый вывод

В ходе диагностики было установлено, что проблема носит составной характер. Замена образа решила проблему с загрузкой, но обнажила проблему с DNS-резолвингом. 
Окончательное решение потребовало исправления **обоих** аспектов: использование современного образа и указание полного DNS-имени сервиса.
