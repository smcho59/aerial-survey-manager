# Sprint Completion Summary

This file maps implemented code to the 1st-4th sprint goals and is the core reference for core-development status.

## 기준

- 기준일: 2026-02-14
- 운영 튜닝(임계치 실측 수치 보정): **다음 턴에서 진행**
- 검토 범위: 사용자/조직/권한, 조직 격리, 배치 처리, 엔진 정책/기본 모니터링(코어 구현)
- 이 문서는 실운영 운영 임계치 최적화 이전의 기준 문서입니다.

## Sprint Execution Status

| Sprint | 이전(요구사항) | 현재 구현 상태 | 완료 여부 |
| --- | --- | --- | --- |
| 1차 | 사용자/조직/권한 API + UI 바인딩 | users/organizations/permissions 라우트와 auth 보호, 관리자 메뉴/버튼/라우트 조건 노출까지 구현 | 완료(핵심 목적 충족) |
| 2차 | 조직 격리 + 쿼터 강제 + 감사 로그 | projects/upload/download/processing 조회·쓰기 경로에 조직 스코프 고정, 조직별 쿼터 + 정책 에러 코드 반환, 조직/권한/배치 변경 감사 이벤트 | 완료(핵심 목적 충족) |
| 3차 | 배치 API + 부분 실패 제어 | `POST /projects/batch` 추가, App.jsx의 반복 호출 삭제 플로우 전환, 실패 항목 반환 및 재시도 UX | 완료(핵심 목적 충족) |
| 4차 | 엔진/큐 정책 정리 + COG 성능 + 운영 모니터링 | 처리 엔진 정책·비활성 엔진 처리, TiTiler 경로/COG 캐시/경고 로그, 큐·처리 지연·메모리 SLO 로깅 | 완료(임계치 튜닝 보류) |

## 구현 목적 대비 보완 대상(핵심)

- 다운로드 토큰 스코프 정책이 현재 “토큰 기반 우선”과 “사용자 인증 기반”이 혼재된 상태입니다.  
  다음 단계에서 정책 한 가지로 통일해야 합니다.
- 그룹 삭제 경로의 정합성(연쇄 정리/배경 작업 연동) 점검이 추가로 필요합니다.
- 조직 내 협업 사용 시, owner 기반 제약이 조직 기준 정책과 충돌할 수 있는 경로가 남아 있습니다.

위 항목은 **기능 구현은 완료됐으나 운영 신뢰성 강화 단계에서 우선 조치 대상**입니다.

## Sprint 1 (Users / Organizations / Permissions API + UI binding)

- Backend APIs
  - `backend/app/api/v1/users.py`
  - `backend/app/api/v1/organizations.py`
  - `backend/app/api/v1/permissions.py`
  - Router registration: `backend/app/api/v1/__init__.py`
- Role/permission UI binding
  - `src/contexts/AuthContext.jsx`
  - Admin tabs and permission-based rendering: `src/App.jsx`, `src/components/Dashboard/Header.jsx`

## Sprint 2 (Org isolation + quota + audit)

- Organization boundary enforcement
  - Scoped access in `projects`, `upload`, `download`, `processing` APIs
- Quota enforcement
  - `backend/app/services/quota.py`
  - Applied on project creation and uploads
- Audit logs
  - `backend/app/utils/audit.py`
  - Applied to user/org/permission changes and project batch actions
- Force organization delete consistency
  - Unassign related FK references before deletion in `backend/app/api/v1/organizations.py`

## Sprint 3 (Batch operations + partial failure UX)

- Batch API
  - `POST /api/v1/projects/batch` in `backend/app/api/v1/projects.py`
  - Supports `delete` / `update_status` with per-item failures
- Frontend integration
  - Sidebar and group actions use batch APIs in `src/App.jsx`
  - Partial failure and retry flows implemented

## Sprint 4 (Engine policy + COG performance + monitoring)

- Engine/runtime policy
  - API/runtime locked to `metashape` for processing start
  - Compose deactivates ODM/external worker services
  - External COG ingest task is disabled by default (`ENABLE_EXTERNAL_COG_INGEST=false`)
- COG performance path
  - TiTiler tile streaming + cache/workers in `src/components/Dashboard/FootprintMap.jsx`
- Monitoring/SLO thresholds
  - COG lookup latency warning (`COG_LOOKUP_WARN_MS`)
  - Queue wait / total time / memory warnings in worker task logs
  - Threshold env wiring in `docker-compose.yml`
  - Ops guide: `docs/OPERATIONS_MONITORING.md`

## Final permission alignment update

- Project responses now include effective permission fields:
  - `current_user_permission`, `can_edit`, `can_delete`
- Frontend project actions now resolve by project-level effective permission (not only global role):
  - `src/App.jsx`
  - `src/components/Dashboard/Sidebar.jsx`

## 업로드 성능 튜닝 기록 (2026-02-26)

- 적용 내역
  - Frontend 업로드 기본값 변경: `src/App.jsx`
    - `partSize`: `10MB -> 32MB`
    - `concurrency`: `6 -> 3`
    - `partConcurrency`: `4 -> 2`
  - Nginx 업로드 경로 버퍼링 튜닝
    - `nginx.conf`
    - `nginx.prod.conf`
    - `/api/v1/upload/` 블록에 다음 적용
      - `proxy_request_buffering off`
      - `proxy_buffering off`
      - `client_max_body_size 0`
      - `proxy_read_timeout 3600s`
      - `proxy_send_timeout 3600s`
      - `proxy_connect_timeout 60s`

- 네트워크 여건별 기대 동작 (예상)
  - `~100 Mbps`(약 `12MB/s` 미만): 현재 튜닝이 병목 완화에 큰 도움을 줄 수 있음
  - `100~300 Mbps`(약 `12~37MB/s`): 병목 완화 + HDD 쓰기/서버 처리 한계 병행 개선
  - `300Mbps+`(약 `37MB/s+`): 디스크 I/O 또는 API 처리 병목이 상대적으로 더 크게 보일 수 있음
  - `iperf3`에서 합산(`SUM`) 업로드가 `138 Mbit/s`이면 실제 체감 `약 17MB/s` 수준과 일치

- 추후 적용 판단 가이드
  - 네트워크가 좋은 환경이면 `concurrency 3 / partConcurrency 2`를 `concurrency 2 / partConcurrency 1`로 낮추는 것도 재시도 포인트
  - HDD 이외 SSD 환경이면 `partSize 64MB`로 늘려도 오히려 유리할 수 있음
  - 현재 값은 먼저 “안전한 기본값(네트워크 병목 완화형)”으로 적용 후, 환경별 성능 측정 기반으로 튜닝

- 적용 범위(운영 판단) 요약
  - 현재 변경은 개발 스택 기준으로 적용되었고, 현 시점 목표는 먼저 안정성 확인이다.
  - 프로덕션 스택은 이슈 해결 이전에 즉시 반영하지 않았다.
  - 운영 반영은 `docker-compose.prod.yml` 기반 배포로 전환할 때, 또는 다음 배포 빌드 단계에서 한 번에 적용한다.

- Nginx 반영/리로드 명령 정리
  - 개발/기본 스택(`docker-compose.yml` 사용 시)
    - `docker compose exec nginx nginx -t`
    - `docker compose exec nginx nginx -s reload`
    - 변경이 안 되면 `docker compose restart nginx`
  - 프로덕션 스택(`docker-compose.prod.yml` 사용 시)
    - `docker compose -f docker-compose.prod.yml exec nginx nginx -t`
    - `docker compose -f docker-compose.prod.yml exec nginx nginx -s reload`
    - 변경이 안 되면 `docker compose -f docker-compose.prod.yml restart nginx`
    - 컨테이너 교체 반영: `docker compose -f docker-compose.prod.yml up -d nginx`

  - 권장 순서
    - config 확인: `nginx -t`
    - 정상일 때 reload
    - 재시작은 서비스가 잠깐 중단되어도 되는 유지보수 시간대에 실행
