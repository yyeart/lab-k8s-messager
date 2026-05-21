# Лабораторная работа: Запуск микросервисного приложения в Kubernetes

## Выполнил
Смирнов Вячеслав Артемович<br>
М8О-106БВ-25

## Описание
В этой лабораторной работе разворачивается микросервисное приложение-мессенджер в Kubernetes кластере

### Используются:
- `Kubernetes` для оркестрации сервисов
- `Kustomize` для разделения конфигов `base`, `dev` и `prod`
- `Argo CD` для GitOps-деплоя
- `S3 CSI` для хранения загружаемых файлов из `messager-service`
  
## Цель работы
Подготовить инфраструктурную конфигурацию для запуска приложения в Kubernetes и продемонстрировать:
- развертывание всех сервисов приложения
- применение миграций базы данных
- подключение файлового хранилища через S3 CSI
- настройка `nodeAffinity`
- поддержку окружений `dev` и `prod` через `kustomize`
- GitOps-синхронизация через Argo CD

## Дополнительная информация

[extra-report](docs/extra-report.md)
- Драйвер
- Сдача по пунктам
- Скриншоты-доказательсва
- Особенности
  
находятся в
```bash
docs/extra-report.md
```
  
## Состав приложения
- `frontend` - SPA интерфейс пользователя
- `bff` - Backend For Frontend, единая точка входа frontend
- `user-service` - сервис пользователей
- `message-service` - сервис сообщений и файлов
- `postgres` - база данных
- `migrate-users` - job для миграций БД пользователей
- `migrate-messages` - job для миграций БД сообщений
- `minio` - хранилище файлов для `message-service`
  
## Структура репозитория
```text
.
├── argocd/                  # Application-манифесты Argo CD
├── bff/                     # исходный код BFF
├── docs/                    # материалы, пояснения и чек-листы
├── frontend/                # frontend-приложение
├── k8s/
│   ├── base/                # базовые Kubernetes-манифесты
│   └── overlays/
│       ├── dev/             # конфигурация для dev
│       └── prod/            # конфигурация для prod
├── message-service/         # сервис сообщений
├── user-service/            # сервис пользователей
├── docker-compose.yml       # локальный запуск приложения
└── README.md
```

## Используемые образы
Для лабораторной используются готовые образы:
- `mablinov2704/frontend:latest`
- `mablinov2704/bff:latest`
- `mablinov2704/user-service:latest`
- `mablinov2704/message-service:latest`

Дополнительно используются:
- `postgres:16-alpine`
- `ghcr.io/kukymbr/goose-docker:latest`
- `minio/minio:latest`
  
## Локальный запуск `dev` через `Minikube`
Предварительно, у вас должен быть установлен `csi-драйвер`: [ch.ctrox.csi.s3-driver](https://github.com/yyeart/csi-s3)

1. Поднимем ноды и зададим им `workload`:
```bash
minikube -p messager start --nodes 3
kubectl label nodes messager workload=system
kubectl label nodes messager-m02 workload=app
kubectl label nodes messager-m03 workload=app disk=fast
```

2. Поднимем `minio` и создадим `bucket`
```bash
kubectl apply -f k8s/overlays/dev/namespace.yaml
kubectl apply -k k8s/base/minio -n messager
kubectl get pods -n messager -w
# ждем пока статус не будет Running
kubectl port-forward svc/minio 9001:9001 -n messager
```
Переходим на `localhost:9001`, авторизуемся через `yyeart:sigma2007`

Создаем bucket под названием `uploads`

3. Поднимем остальные сервисы
```bash
kubectl apply -k k8s/overlays/dev
kubectl get pods -n messager -w
```
Все `pods` должны быть `Running/Complete`

### Для запуска `prod` алгоритм аналогичен:
- `Kustomize`, `namespace` применяем из `k8s/overlays/prod`
- соответственно вместо `-n messager` используем `-n messager-prod`
- `bucket` в `minio` называем `uploads-prod`

## S3 (MinIO + CSI)
Для `message-service` загрузка файлов должна работать через смонтированное S3-хранилище

Ожидаемое поведение:
- сервис пишет файлы не на локальный volume, а в каталог, подключенный через CSI
- после загрузки  файл появится в bucket
- путь к каталогу задается через `UPLOADS_DIR`

## Node Affinity
В лабораторной настроено размещение сервисов по узлам:
- `postgres` и `minio` - на узлах с метой `workload=system`
- прикладные сервисы - на узлах с меткой `workload=app`
- для `message-service` дополнительно задается предпочтение `disk=fast`
  
## Argo CD
В каталоге `argocd/` находятся Application-манифесты:
- application_dev.yaml
- application_prod.yaml
  
Они настраивают GitOps-деплой из репозитория в `Kubernetes` через `Argo CD`

### Установка и применение
1. Установим оффициальные `pods`
```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
2. Прокинем на порт и авторизуемся
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Получаем пароль
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode; echo
```
Переходим на `localhost:8080` и авторизуемся через admin:`password`

3. Применяем манифесты
```bash
kubectl apply -f argocd/application_dev.yaml
kubectl apply -f argocd/application_prod.yaml
```
4. Ожидаемый статус в интерфейсе Argo CD:
- статус `Synced`
- статус `Healthy`

## Доступ к приложению
```bash
minikube -p messager service frontend -n messager --url
```