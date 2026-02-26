# Operations Monitoring (Sprint 4)

이 문서는 4차 스프린트 기준으로 수집되는 운영 신호를 정리합니다.  
임계치 기본값은 기본 동작 상태이며, 실제 운영 부하 기반 최적화는 다음 턴에서 보강합니다.

This document defines runtime thresholds and log signals used for processing and COG access monitoring.

## Environment variables

Set these in `.env` (or keep defaults from `docker-compose.yml`):

- `COG_LOOKUP_WARN_MS` (default: `1000`)
  - Warn when `/api/v1/download/projects/{project_id}/cog-url` lookup latency exceeds this value.
- `PROCESSING_QUEUE_WAIT_WARN_SECONDS` (default: `300`)
  - Warn when processing job queue wait time exceeds this value.
- `PROCESSING_TOTAL_WARN_SECONDS` (default: `7200`)
  - Warn when total processing duration exceeds this value.
- `PROCESSING_MEMORY_WARN_MB` (default: `8192`)
  - Warn when worker max RSS exceeds this value.
- `ENABLE_EXTERNAL_COG_INGEST` (default: `false`)
  - Keeps external COG ingest task disabled unless explicitly enabled.

## Log signals

- COG lookup latency warning:
  - Logger: `backend.app.api.v1.download`
  - Pattern: `cog_lookup_slow project_id=... lookup_ms=... threshold_ms=...`

- Processing SLO warnings:
  - Source: `backend/app/workers/tasks.py`
  - Pattern: `[SLO][WARN] ...`
  - Includes:
    - queue wait exceeded
    - total elapsed exceeded
    - memory exceeded

- Processing completion metrics payload:
  - Returned in Celery task result (`process_orthophoto`) under `metrics`
  - Includes:
    - `queue_wait_seconds`
    - `total_elapsed_seconds`
    - `memory_usage_mb`
    - `slo` thresholds and exceed flags
    - per-phase elapsed timings

## Policy notes

- Runtime processing engine support follows `ENABLE_*_ENGINE` policy flags:
  - `ENABLE_METASHAPE_ENGINE`
  - `ENABLE_ODM_ENGINE`
  - `ENABLE_EXTERNAL_ENGINE`
- APIs return explicit `unsupported_engine` errors when a disabled or unknown engine is requested.
- External COG ingest remains disabled by default via `ENABLE_EXTERNAL_COG_INGEST=false` and can be enabled only intentionally.

## Policy change checklist

1. 엔진 정책 값 수정
   - `.env`의 `ENABLE_METASHAPE_ENGINE`, `ENABLE_ODM_ENGINE`, `ENABLE_EXTERNAL_ENGINE` 값을 변경합니다.
   - 변경 목적(비활성화/활성화)과 대상 환경(운영/개발)을 기록합니다.
2. 서비스 적용
   - 백엔드/워커를 재시작합니다: `docker compose up -d --force-recreate backend` (필요 시 worker/queue도 동일).
   - 기존 Celery 작업이 계속 쌓여 있으면 상태를 확인하고 필요 시 정리합니다.
3. 정책 반영 검증
   - 프론트가 사용할 엔진 목록을 확인: `GET /api/v1/processing/engines`
   - 비활성 상태로 둔 엔진의 시작 요청 시 `unsupported_engine`(400) 응답이 반환되는지 확인합니다.
4. 후속 보완
   - `/docker-compose.yml`의 서비스 주석/주석 해제 정책은 정책값과 함께 문서와 일치시킵니다.

## 업로드 성능 튜닝 메모 (2026-02-26)

### 적용 기준

- 최근 이슈 대응으로 업로드 기본값 조정이 이루어졌다.
  - `partSize`: `10MB -> 32MB`
  - `concurrency`: `6 -> 3`
  - `partConcurrency`: `4 -> 2`
- 현재 이 값은 코드(프론트엔드)와 Nginx 업로드 경로(`/api/v1/upload/`) 설정에 반영되어 있다.
- 프로덕션 스택은 현재 즉시 반영되지 않았고, 배포 타이밍에 맞춰 적용한다.

### 성능 기대치

- `~100 Mbps`: 현재 값에서 네트워크 대기/병목 완화 효과가 큼.
- `100~300 Mbps`: 병목 완화 + HDD 쓰기 성능 영향이 함께 보임.
- `300 Mbps 이상`: HDD 쓰기 한계/시스템 부하가 더 큰 병목으로 보일 수 있음.

### 운영 체크 항목

- 업로드 실패율(HTTP 4xx/5xx), 재시도 횟수 상승 여부
- `/api/v1/upload/` 경로에서 Nginx 재연결/타임아웃 로그
- 대상 서버 저장 장치 사용률(특히 HDD 사용 환경)

### 환경별 튜닝 전환 가이드

- 네트워크 여유가 큰 환경에서 업로드가 여전히 느리면:
  - `partConcurrency`를 `1`로 낮추거나
  - `concurrency`를 `2`로 낮춰 시스템 안정성 우선순위 적용
- SSD 환경에서는 `partSize`를 `64MB`로 상향 테스트를 검토할 수 있음.

## Queue & worker diagnostics

- Redis 큐 잔여량
  - `docker compose exec redis redis-cli llen metashape`
  - `docker compose exec redis redis-cli llen odm`
  - `docker compose exec redis redis-cli llen external`
  - 큐 길이가 계속 증가하면 워커 스케일/장애 여부를 의심합니다.
- 워커 프로세스 상태
  - `docker compose ps`로 backend/worker 컨테이너 상태 점검
  - `docker compose logs -f worker --tail=200`로 최근 처리 로그 확인
  - (운영 환경) 큐별 active worker 목록/태스크 수를 Celery 모니터로 확인
- 정책 불일치 대응
  - 엔진이 ON인데 해당 큐 처리량이 0인 경우: 워커 재시작 or 큐 네임 불일치 여부 점검
  - 큐만 쌓이고 처리되지 않으면 Redis 연결, 환경변수 `CELERY_BROKER_URL`, queue_name 매핑(`metashape/odm/external`) 점검
  - 엔진 라우트 정책이 UI와 다르면 `/api/v1/processing/engines`, docker startup log, 배포 `.env`를 동기화합니다.

### 빠른 진단 실행

- `scripts/check-processing-ops.sh` 실행으로 정책/큐/워커 상태를 한 번에 점검
- 실행 방법:
  - `chmod +x scripts/check-processing-ops.sh`
  - `./scripts/check-processing-ops.sh`
- 점검 항목:
  - `.env` 엔진 플래그(`ENABLE_*_ENGINE`, `ENABLE_EXTERNAL_COG_INGEST`)
  - `/api/v1/processing/engines` 응답
  - Redis 큐 backlog (`metashape`, `odm`, `external`)
  - 핵심 서비스 상태 및 celery ping
