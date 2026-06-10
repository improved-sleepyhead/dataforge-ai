# DataForge AI: запуск MVP

Эта заметка описывает, как собрать локальный polyrepo-workspace и запустить
MVP. Корневая папка нужна только для координации; каждый сервис живет в своем
репозитории.

## Репозитории проекта

### Frontend

```text
https://github.com/improved-sleepyhead/dataforge-ai-frontend
```

### Backend

```text
https://github.com/improved-sleepyhead/dataforge-ai-backend
```

### ML Service

```text
https://github.com/improved-sleepyhead/dataforge-ai-ml-service
```

### Deploy settings

```text
https://github.com/improved-sleepyhead/dataforge-ai-deploy
```

## Клонирование

```bash
mkdir -p dataforge
cd dataforge

git clone https://github.com/improved-sleepyhead/dataforge-ai-deploy dataforgeai-deploy
git clone https://github.com/improved-sleepyhead/dataforge-ai-frontend dataforgeai-frontend
git clone https://github.com/improved-sleepyhead/dataforge-ai-backend dataforgeai-backend
git clone https://github.com/improved-sleepyhead/dataforge-ai-ml-service dataforge-ai-ml-service
```

Ожидаемая структура:

```text
dataforgeai-deploy/
dataforgeai-frontend/
dataforgeai-backend/
dataforge-ai-ml-service/
```

## Кратко об инфраструктуре

- `dataforgeai-frontend`: Next.js UI, доступен пользователю в браузере.
- `dataforgeai-backend`: NestJS control plane. Владеет проектами, датасетами,
  job records, approvals, audit, Prisma/PostgreSQL metadata и signed URLs.
- `dataforge-ai-ml-service`: приватный FastAPI/Dagster compute plane. Владеет
  профилированием, evidence, reports, ActionPlan artifacts, exports и compute
  events.
- Object storage: MinIO/S3-compatible слой для raw uploads и generated
  artifacts. Сервисы передают `ArtifactRef`, а не raw blobs.
- `dataforgeai-deploy`: Helm chart, env values, Jenkins deploy skeleton,
  NetworkPolicy, smoke checks, rollback/security docs.

Публичный доступ есть только к frontend/backend. ML service, Dagster,
PostgreSQL, MinIO internal endpoint, workers, Qdrant/Milvus и MLflow остаются
приватными.

## Локальный запуск сервисов

Запускайте каждый сервис в отдельном терминале.

### Frontend

```bash
cd dataforgeai-frontend
pnpm install
NEXT_PUBLIC_API_MODE=mock pnpm dev
```

Открыть:

```text
http://localhost:3000
```

Чтобы подключить frontend к локальному backend:

```bash
NEXT_PUBLIC_API_MODE=real NEXT_PUBLIC_API_BASE_URL=http://localhost:4200/api/v1 pnpm dev
```

### Backend

```bash
cd dataforgeai-backend
npm install
npm run prisma:validate
npm run prisma:generate
DATABASE_URL="postgresql://user:password@localhost:5432/dataforgeai" npm run start:dev
```

Backend по умолчанию слушает `4200`, если не задан `PORT`.

Для реальных metadata-сценариев сначала поднимите PostgreSQL и примените
локальные dev-миграции:

```bash
npm run prisma:migrate:dev
```

### ML service

```bash
cd dataforge-ai-ml-service
python -m venv .venv
PYTHON=.venv/bin/python make install-dev
.venv/bin/python -m uvicorn app.api.main:app --host 127.0.0.1 --port 8000
```

Полезные endpoint'ы:

```text
GET  http://127.0.0.1:8000/api/v1/health
GET  http://127.0.0.1:8000/api/docs
```

MVP compute demo без реального backend:

```bash
cd dataforge-ai-ml-service
PYTHON=.venv/bin/python make run-mvp-demo
```

## Kubernetes rehearsal

Используйте этот путь, когда готовы service images и локальные runtime secrets:

```bash
cd dataforgeai-deploy
jenkins/scripts/validate-values.sh local envs/local/values.example.yaml
jenkins/scripts/render-manifests.sh local
```

Dry-run:

```bash
helm upgrade --install dataforgeai ./helm/dataforgeai \
  --namespace dataforgeai-local \
  --create-namespace \
  --dry-run \
  -f envs/local/values.example.yaml
```

Apply в minikube/kind:

```bash
helm upgrade --install dataforgeai ./helm/dataforgeai \
  --namespace dataforgeai-local \
  --create-namespace \
  --atomic \
  --timeout 10m \
  -f envs/local/values.example.yaml
```

Smoke skeleton:

```bash
RUN_CLUSTER_CHECKS=true jenkins/scripts/smoke-local.sh
```

Если cluster/images/secrets не подготовлены, smoke script вернет
`SKIPPED_CLUSTER`, а не ложный pass.

## Runtime secrets

Не коммитьте реальные значения. Для Kubernetes rehearsal нужны runtime Secret
или ExternalSecret записи для:

- PostgreSQL credentials backend;
- object-storage app access credentials;
- optional MinIO bootstrap/admin credentials только для local/dev;
- backend-to-ML service authentication/signature material.

Object-storage non-secret env names рендерятся chart'ом:

```text
OBJECT_STORAGE_ENDPOINT
OBJECT_STORAGE_BUCKET
OBJECT_STORAGE_REGION
OBJECT_STORAGE_FORCE_PATH_STYLE
DATAFORGE_OBJECT_STORAGE_ENDPOINT_URL
DATAFORGE_OBJECT_STORAGE_BUCKET
DATAFORGE_OBJECT_STORAGE_REGION
DATAFORGE_OBJECT_STORAGE_PREFIX_ROOT
```

## Правила безопасности

- Не хранить реальные secrets или customer PII в репозиториях.
- Demo data только synthetic/fake.
- Production не использует floating `latest`.
- Stage/prod используют managed/HA PostgreSQL и controlled/HA MinIO.
- ML service приватный и вызывается backend, не браузером.
- Raw files и generated artifacts передаются через object-storage references.
