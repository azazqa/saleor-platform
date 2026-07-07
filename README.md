![Saleor Platform](https://user-images.githubusercontent.com/249912/71523206-4e45f800-28c8-11ea-84ba-345a9bfc998a.png)

<div align="center">
  <h1>Saleor Platform</h1>
</div>

<div align="center">
  <p>커스텀 Saleor API·Dashboard를 Docker로 실행하고, Storefront는 별도로 구동합니다.</p>
</div>

## 프로젝트 구조

```
saleor-platform/
├── docker-compose.yml   # API, Dashboard, Worker, DB, Cache …
├── backend.env
├── common.env
├── saleor/              # Saleor Core (API, Worker)
├── dashboard/           # Saleor Dashboard
└── storefront/          # Storefront (로컬 dev / 스테이징·운영은 독립 배포)
```

| 디렉터리 | 역할 | 실행 방식 | 포트 |
|---------|------|----------|------|
| `saleor/` | GraphQL API, Celery Worker | Docker Compose | 8000 |
| `dashboard/` | 관리자 대시보드 | Docker Compose | 9000 |
| `storefront/` | 고객용 스토어프론트 | **로컬:** `pnpm dev` / **스테이징·운영:** 독립 서비스 | 3000 |

Storefront는 [saleor-platform](https://github.com/saleor/saleor-platform)과 달리 Compose에 포함하지 않습니다.

## 사전 요구 사항

**Platform (Docker)**

- [Docker](https://docs.docker.com/install/)
- [Docker Compose](https://docs.docker.com/compose/)

**Storefront (로컬 개발)**

- Node.js (storefront `package.json` engines 참고)
- [pnpm](https://pnpm.io/)

## 프로젝트 추가

```shell
git clone <your-saleor-fork-url> saleor
git clone <your-dashboard-fork-url> dashboard
git clone <your-storefront-fork-url> storefront
```

## Platform 실행 (API + Dashboard)

1. **macOS / Windows:** Docker File sharing에 이 디렉터리 추가, 메모리 5 GB 이상

2. 빌드 및 기동:

```shell
docker compose build
docker compose run --rm api python3 manage.py migrate
docker compose run --rm api python3 manage.py populatedb --createsuperuser
docker compose up
```

| 서비스 | URL |
|--------|-----|
| Saleor Core (API) | http://localhost:8000 |
| Saleor Dashboard | http://localhost:9000/dashboard/ |
| Jaeger UI | http://localhost:16686 |
| Mailpit | http://localhost:8025 |

## Storefront (로컬 개발)

Platform(API)이 실행 중인 상태에서:

```shell
cd storefront
cp .env.example .env.local
```

`.env.local` 최소 설정:

```env
NEXT_PUBLIC_SALEOR_API_URL=http://localhost:8000/graphql/
NEXT_PUBLIC_STOREFRONT_URL=http://localhost:3000
NEXT_PUBLIC_DEFAULT_CHANNEL=default-channel
```

```shell
pnpm install
pnpm dev
```

→ http://localhost:3000

로컬 dev 모드(`pnpm dev`)에서는 Next.js가 호스트에서 API(`localhost:8000`)에 접근하므로 **이미지 최적화(`/_next/image`)가 정상 동작**합니다.

`.env.development`에 K8s용 `SALEOR_INTERNAL_API_URL`이 있으므로, 로컬에서는 `.env.local`에 아래를 반드시 설정하세요:

```env
NEXT_PUBLIC_SALEOR_API_URL=http://localhost:8000/graphql/
SALEOR_INTERNAL_API_URL=http://localhost:8000/graphql/
```

## Storefront (스테이징 / 운영)

Storefront는 Platform Compose와 분리된 **독립 서비스**로 배포합니다.

- `storefront/Dockerfile`로 컨테이너 이미지 빌드 후 배포, 또는
- Vercel / K8s 등 별도 호스팅

환경 변수 예 (스테이징):

```env
NEXT_PUBLIC_SALEOR_API_URL=https://api.staging.example.com/graphql/
NEXT_PUBLIC_STOREFRONT_URL=https://shop.staging.example.com
NEXT_PUBLIC_DEFAULT_CHANNEL=default-channel
# K8s 등 내부 API가 다른 경우
SALEOR_INTERNAL_API_URL=http://saleor-api:8000/graphql/
```

Platform `api` 서비스의 `STOREFRONT_URL`을 스테이징/운영 storefront URL로 맞춰 주세요.

## 부분 실행

```shell
docker compose up api worker          # 백엔드만
docker compose up api worker dashboard
```

## 트러블슈팅

```shell
docker compose stop && docker compose rm && docker compose build
docker compose down --volumes db      # DB 초기화 (데이터 삭제)
docker system prune                   # Docker 캐시 정리
```

## 라이선스

[saleor-platform LICENSE](LICENSE)를 따릅니다.
