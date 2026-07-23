![Saleor Platform](https://user-images.githubusercontent.com/249912/71523206-4e45f800-28c8-11ea-84ba-345a9bfc998a.png)

<div align="center">
  <h1>Saleor Platform</h1>
</div>

<div align="center">
  <p>커스텀 Saleor API·Dashboard를 Docker로 실행하고, Storefront·Toss Payments 앱은 별도로 구동합니다.</p>
</div>

## 프로젝트 구조

```
saleor-platform/
├── docker-compose.yml            # 공통 (API, Dashboard, Worker, DB, Cache …)
├── docker-compose.override.yml   # 로컬 전용 (자동 병합)
├── docker-compose.prod.yml       # 운영 전용 (-f 명시)
├── backend.env                   # Saleor backend (no PUBLIC_URL — per-service URLs)
├── common.env
├── haproxy/                      # Production L7 routing (HAProxy)
├── saleor/                       # Saleor Core (API, Worker)
├── saleor-dashboard/             # Saleor Dashboard
├── tosspayments-app/             # Toss Payments Saleor App
└── storefront/                   # Storefront (로컬 dev / 스테이징·운영은 독립 배포)
```

| 디렉터리 | 역할 | 실행 방식 | 포트 |
|---------|------|----------|------|
| `saleor/` | GraphQL API, Celery Worker | Docker Compose | 8000 |
| `dashboard/` | 관리자 대시보드 | Docker Compose | 9000 |
| `tosspayments-app/` | Toss Payments 앱 | **로컬:** `pnpm dev` / **운영:** 독립 서비스 | 3001 |
| `storefront/` | 고객용 스토어프론트 | **로컬:** `pnpm dev` / **스테이징·운영:** 독립 서비스 | 3000 |

Storefront·Toss Payments 앱은 [saleor-platform](https://github.com/saleor/saleor-platform)과 달리 Compose에 포함하지 않습니다. 호스트에서 `localhost` URL로 통일해 Docker 내부 DNS(`api`, `tosspayments-app`)와의 불일치를 피합니다.

## 사전 요구 사항

**Platform (Docker)**

- [Docker](https://docs.docker.com/install/)
- [Docker Compose](https://docs.docker.com/compose/)

**Storefront / Toss Payments App (로컬 개발)**

- Node.js (`package.json` engines 참고)
- [pnpm](https://pnpm.io/)

## 프로젝트 추가

```shell
git clone <your-saleor-fork-url> saleor
git clone <your-dashboard-fork-url> dashboard
git clone <your-storefront-fork-url> storefront
```

## Platform 실행 (API + Dashboard)

Compose 파일 역할:

| 파일 | 역할 | 사용 |
|------|------|------|
| `docker-compose.yml` | 공통 | 항상 포함 |
| `docker-compose.override.yml` | 로컬 (`ENABLE_SSL=False`, localhost URL) | `docker compose up` 시 **자동** 병합 |
| `docker-compose.prod.yml` | 운영 (`ENABLE_SSL=True`, 공인 URL) | `-f`로 **명시** |

1. **macOS / Windows:** Docker File sharing에 이 디렉터리 추가, 메모리 5 GB 이상

2. **로컬** 빌드 및 기동 (`override.yml` 자동 적용):

```shell
docker compose build
docker compose run --rm api python3 manage.py migrate
docker compose run --rm api python3 manage.py populatedb --createsuperuser
docker compose up
```

3. **운영** 기동 (override는 적용되지 않음; 재생성·재시작에도 동일 `-f` 사용):

```shell
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
# 예: api·worker·dashboard 재생성
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --force-recreate api worker dashboard
```

| 서비스 | URL |
|--------|-----|
| Saleor Core (API) | http://localhost:8000 |
| Saleor Dashboard | http://localhost:9000/dashboard/ |
| Jaeger UI | http://localhost:16686 |
| Mailpit | http://localhost:8025 |

Toss Payments App·Storefront는 아래 절을 참고해 **호스트에서 별도 실행**합니다 (각각 http://localhost:3001, http://localhost:3000).

## URL 전략 (서비스별 고정 URL)

`PUBLIC_URL`은 사용하지 않습니다. 각 서비스가 **자기 URL**로 연결됩니다.

| 서비스 | 로컬 (`override.yml`) | 운영 (`prod.yml` / HAProxy) |
|--------|------------------------|------------------------------|
| API | `http://localhost:8000/graphql/` | `https://saleor-api.klms.co.kr/graphql/` |
| Dashboard | `http://localhost:9000/dashboard/` | `https://saleor-dashboard.klms.co.kr/dashboard/` |
| Storefront | `http://localhost:3000/` | `https://saleor.klms.co.kr/` |
| Toss Payments App | `http://localhost:3001` | `https://tosspayments.klms.co.kr` |

Saleor가 앱 등록·절대 URL에 쓰는 API 호스트는 Django **Site** 도메인으로 결정됩니다 (`PUBLIC_URL` 미설정 시). 운영에서는 Site domain을 `saleor-api.klms.co.kr`로 맞추세요.

### `ENABLE_SSL` (api·worker)

웹훅·절대 URL의 http/https 스킴은 `api`·`worker`의 `ENABLE_SSL`로 결정됩니다.

| 환경 | 파일 | `ENABLE_SSL` | 생성되는 스킴 |
|------|------|--------------|----------------|
| 로컬 개발 | [`docker-compose.override.yml`](docker-compose.override.yml) | `False` | `http://` |
| 운영 (HAProxy TLS) | [`docker-compose.prod.yml`](docker-compose.prod.yml) | `True` | `https://` |

- 운영에서 `False`면 Saleor가 `http://saleor-api.klms.co.kr/graphql/`로 웹훅 `Saleor-Api-Url`을 만들어, `https`로 등록된 앱(Toss)이 auth 데이터를 못 찾고 `NOT_REGISTERED`(HTTP 401)가 납니다.
- `api`와 `worker`는 **항상 같은 값**으로 맞추세요. (sync 웹훅은 `api`, async는 `worker`가 전송)
- 값 변경 후에는 `--force-recreate`로 **재생성**해야 반영됩니다 (restart로는 env 미반영). 운영은 `-f docker-compose.yml -f docker-compose.prod.yml`을 함께 쓰세요.

### 환경 변수 (tosspayments-app, 로컬)

앱은 호스트에서 `pnpm dev`로 실행합니다. **브라우저·Saleor 웹훅 auth**는 `localhost`로 통일합니다.

| 변수 | 로컬 (호스트 `pnpm dev`) | 운영 |
|------|--------------------------|------|
| `SALEOR_API_URL` | `http://localhost:8000/graphql/` | `https://saleor-api.klms.co.kr/graphql/` |
| `APP_IFRAME_BASE_URL` | `http://localhost:3001` | `https://tosspayments.klms.co.kr` |
| `APP_API_BASE_URL` | `http://host.docker.internal:3001` | `https://tosspayments.klms.co.kr` |

`APP_API_BASE_URL`은 Saleor API(Docker)가 호스트에서 돌아가는 앱으로 **웹훅**을 보낼 때 쓰는 URL입니다. 로컬 [`docker-compose.override.yml`](docker-compose.override.yml)의 `api`/`worker`에 `host.docker.internal`이 설정되어 있어야 합니다.

등록 시 [`register.ts`](tosspayments-app/src/pages/api/register.ts)는 `SALEOR_API_URL`을 auth 키로 사용합니다. Dashboard·Storefront와 동일하게 `http://localhost:8000/graphql/`을 쓰세요.

`HTTP_IP_FILTER_ENABLED`: 로컬 [`docker-compose.override.yml`](docker-compose.override.yml) `False` / 운영 [`docker-compose.prod.yml`](docker-compose.prod.yml) `True` (`api`·`worker` 동일).

### 운영 HAProxy

[`haproxy/haproxy.cfg`](haproxy/haproxy.cfg) — `saleor-api.klms.co.kr`, `saleor-dashboard.klms.co.kr`, `tosspayments.klms.co.kr`

자세한 배포: [`haproxy/README.md`](haproxy/README.md)

## Toss Payments 연동

### 1. Payment App (로컬)

Platform(API)이 실행 중인 상태에서:

```shell
cd tosspayments-app
cp .env.example .env
# .env — TOSS_CLIENT_KEY, TOSS_SECRET_KEY, SALEOR_API_URL=http://localhost:8000/graphql/
pnpm install
pnpm dev
```

→ http://localhost:3001

URL·키는 [`tosspayments-app/.env.example`](tosspayments-app/.env.example)를 따릅니다.

### 2. Dashboard 앱 설치

1. `docker compose up` 및 `pnpm dev`(tosspayments-app) 실행
2. Dashboard → **Apps** → **Install from manifest**
3. Manifest URL (Saleor API 컨테이너 → 호스트의 앱):

```
http://host.docker.internal:3001/api/manifest
```

`localhost:3001`은 API 컨테이너 기준으로 API 자신을 가리킵니다. `tosspayments-app:3001`은 Compose에 앱이 없으므로 사용하지 마세요.

4. 권한: `HANDLE_PAYMENTS`, `MANAGE_CHECKOUTS`

Gateway ID: `klms.app.payment.tosspayments`

**설치·웹훅 실패 시:** `.env`의 `SALEOR_API_URL`이 `http://localhost:8000/graphql/`인지, `APP_API_BASE_URL`이 `http://host.docker.internal:3001`인지 확인하세요. URL을 바꾼 뒤에는 `tosspayments-app/.auth-data.json`을 삭제하고 앱을 재설치하세요.

### 3. Storefront env

```env
NEXT_PUBLIC_ENABLE_TOSS_PAYMENTS=true
ENABLE_TOSS_PAYMENTS=true
```

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
