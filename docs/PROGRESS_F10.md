# 작업 완료 기록 — F10

## F10: 인스펙터 레이아웃 개선 및 프로젝트 메타 정보 강화 - 2026-03-16
### 요구사항
1. COG 썸네일 해상도 증가 — gdal_translate `-outsize 1024` → `4096`
2. 저장용량 표시 수정 — `du -sb` 전체 디렉토리 → DB `ortho_size` 합산 + tiles 디렉토리
3. SLO 성능 지표 섹션 제거
4. "촬영일" → "처리완료일" 변경
5. InspectorPanel 3칸 → 2칸 레이아웃 변경 (프로젝트 정보 2/3, 정사영상 1/3)
   - 프로젝트 정보 + 원본 데이터를 2열 서브컬럼으로 배치
   - 정사영상 썸네일 `object-cover` → `object-contain`, `h-80`
6. 프로젝트 전환 시 EO 마커 상태 누출 수정
7. 처리 모드 한글화: Normal→정밀 처리, Fast→고속 처리, Preview→미리보기, High→고정밀 처리
8. 프로젝트 생성일, 처리완료일, 처리 소요시간 추가 (backend: `processing_started_at`, `processing_completed_at` 필드)
9. 인스펙터 열릴 때 지도 높이 자동 축소 (1000px → 900px)
10. 저장용량 실시간 반영: projects 변경 시 자동 갱신

### 진행 체크리스트
- [x] A1. 계획 수립
- [x] A2. 계획 검토
- [x] A3. 과도 설계 검토
- [x] 🚪 사용자 승인
- [x] B1. 구현
- [x] B2. 목적 부합 검토
- [x] B3. 버그/보안 검토 및 수정
- [x] B4. 수정사항 재검토
- [x] C1. 파일/함수 분리
- [x] C2. 코드 통합/재사용 검토
- [x] C3. 사이드이펙트 확인
- [x] C4. 불필요 코드 정리
- [x] C5. 코드 품질 검토
- [x] 🚪 변경사항 보고
- [x] D1. UX 관점 검토
- [x] D2. 전체 변경사항 통합 검토
- [x] D3. 배포 가능성 판단
- [x] 🚪 최종 승인
- [x] D4. 커밋

### 구현 계획

**변경 1: COG 썸네일 해상도 증가**
- `gdal_translate -outsize` 값을 1024에서 4096으로 변경
- 썸네일 품질 향상으로 인스펙터 패널에서 정사영상 미리보기 선명도 개선

**변경 2: 저장용량 표시 수정**
- 기존: `du -sb`로 전체 디렉토리 크기 계산 (느리고 부정확)
- 변경: DB `ortho_size` 합산 + tiles 디렉토리 크기로 계산
- `backend/app/api/v1/projects.py` 수정

**변경 3: SLO 성능 지표 섹션 제거**
- 대시보드에서 불필요한 SLO 성능 지표 UI 제거

**변경 4: "촬영일" → "처리완료일" 변경**
- `src/components/Dashboard/Sidebar.jsx`: 라벨 변경
- `src/App.jsx`: `completedDate` 매핑 추가

**변경 5: InspectorPanel 레이아웃 개선**
- 3칸 → 2칸 레이아웃 (프로젝트 정보 2/3, 정사영상 1/3)
- 프로젝트 정보 영역을 2열 서브컬럼으로 구성 (프로젝트 정보 + 원본 데이터)
- 정사영상 썸네일: `object-cover` → `object-contain`, 높이 `h-80`
- `src/components/Project/InspectorPanel.jsx` 수정

**변경 6: EO 마커 상태 누출 수정**
- 프로젝트 전환 시 이전 프로젝트의 EO 마커가 남아있는 문제 해결

**변경 7: 처리 모드 한글화**
- `InspectorPanel.jsx`: 처리 모드 라벨 매핑 추가
  - Normal → 정밀 처리, Fast → 고속 처리, Preview → 미리보기, High → 고정밀 처리

**변경 8: 처리 시간 정보 추가**
- `backend/app/schemas/project.py`: `processing_started_at`, `processing_completed_at` 필드 추가
- `backend/app/api/v1/projects.py`: 처리 시간 필드 반환 로직 추가
- `src/App.jsx`: `createdDate`, `processingStartedAt`, `processingCompletedAt` 매핑
- `src/components/Project/InspectorPanel.jsx`: 생성일, 처리완료일, 처리 소요시간 표시

**변경 9: 인스펙터 열릴 때 지도 높이 자동 축소**
- `src/components/Dashboard/DashboardView.jsx`: 인스펙터 패널 표시 시 지도 높이 1000px → 900px

**변경 10: 저장용량 실시간 반영**
- `src/components/Dashboard/DashboardView.jsx`: `projects` 상태 변경 시 저장용량 자동 재계산

### 변경 파일 목록
- `backend/app/api/v1/projects.py` — 처리 시간 필드 반환, 저장용량 계산 방식 변경
- `backend/app/schemas/project.py` — `processing_started_at`, `processing_completed_at` 스키마 필드 추가
- `backend/app/models/project.py` — (기존 필드 활용, 변경 없음)
- `src/App.jsx` — `createdDate`, `processingStartedAt`, `processingCompletedAt` 매핑
- `src/components/Project/InspectorPanel.jsx` — 2칸 레이아웃, 처리 모드 한글화, 처리 소요시간, 썸네일 스타일
- `src/components/Dashboard/DashboardView.jsx` — 지도 높이 자동 축소, 저장용량 실시간 갱신
- `src/components/Dashboard/Sidebar.jsx` — "처리완료일" 라벨 변경

### 회귀 기록
(없음)

### 발견된 이슈 및 결정사항
(없음)

---

## F10-B: 안정성 개선 및 UX 정리 - 2026-03-17

### 요구사항
1. 도엽 패널에서 머지(Merge) 기능 제거
2. 처리 중 단계별 진행 상태 표시 (Sidebar)
3. Celery 이중 실행 버그 수정 (visibility_timeout + 멱등성)
4. 대용량 프로젝트(2000장, 24시간+) 대비 안정성 강화
5. 단일 계정 운영 체제로 변경 (관리 메뉴 제거, 회원가입 제거)
6. 처리 단계 UI: 내보내기를 COG 변환에 통합 표시
7. 프로젝트 3번 DB 상태 복구 (error → completed)

### 구현 내용

**변경 1: 도엽 패널 머지 기능 제거**
- `src/components/Dashboard/FootprintMap.jsx`: 머지 버튼, `handleMerge`, `isMerging` 상태, `Merge` import 제거
- 클립 버튼 `flex-1` → `w-full` (전체 너비)

**변경 2: 처리 단계별 진행 표시**
- `backend/app/schemas/project.py`: `ProcessingJobResponse`에 `step_status` 필드 추가
- `backend/app/api/v1/processing.py`: `_read_step_status_file()` 헬퍼 추가, `get_processing_status` 응답에 포함
- `src/components/Dashboard/Sidebar.jsx`: `ProcessingSteps` 컴포넌트 추가 (5초 polling, 단계별 색상 점)
  - 내보내기(Export Raster)를 COG 변환에 합산 표시 (`mergeKeys` 로직)

**변경 3: Celery 이중 실행 버그 수정**
- `backend/app/workers/tasks.py`:
  - `broker_transport_options={'visibility_timeout': 604800}` (7일) — Redis 재전달 방지
  - `worker_prefetch_multiplier=1` — task 1개씩만 가져와 in-flight 타이머 방지
  - `process_orthophoto`에 `acks_late=True` — 완료 후 ack, worker 크래시 시 안전 재시도
  - 멱등성 체크 추가 — `job.status == 'completed'`이면 재실행 skip
- 근본 원인: Redis visibility_timeout(1시간) 기본값 + prefetch=4로 인해 대기 중인 task가 in-flight 상태로 카운트되어 1시간 초과 시 재전달됨

**변경 4: 프로젝트 3번 DB 복구**
- processing_job `bddfc599`: status `error` → `completed`, started_at/completed_at 정상화, result_size 복원
- project `e9bc76b0`: status `cancelled` → `completed`

**변경 5: 단일 계정 운영 체제**
- `src/components/LoginPage.jsx`: 회원가입 토글/버튼 제거, "이메일" → "아이디", 테스트계정 안내 제거
- `src/components/Dashboard/Header.jsx`: 관리 버튼, 역할 뱃지 제거
- `backend/app/api/v1/auth.py`: `/register` 엔드포인트 → 403 반환
- DB: `admin@siqms.or.kr` / `siqms` 계정 생성 (role=admin), 기존 test 계정 비활성화

### 변경 파일 목록
- `backend/app/workers/tasks.py` — visibility_timeout, prefetch_multiplier, acks_late, 멱등성 체크
- `backend/app/schemas/project.py` — step_status 필드
- `backend/app/api/v1/processing.py` — _read_step_status_file, step_status 응답 포함
- `backend/app/api/v1/auth.py` — /register 비활성화
- `src/components/Dashboard/Sidebar.jsx` — ProcessingSteps 컴포넌트, Export Raster+COG 통합 표시
- `src/components/Dashboard/FootprintMap.jsx` — 머지 기능 제거
- `src/components/Dashboard/Header.jsx` — 관리 메뉴, 역할 뱃지 제거
- `src/components/LoginPage.jsx` — 회원가입 제거, 단일 로그인 폼
- `src/contexts/AuthContext.jsx` — 에러 메시지 객체 → 문자열 변환 처리
- `backend/app/schemas/user.py` — `EmailStr` → `str` 전체 교체 (단순 ID 로그인 지원)
