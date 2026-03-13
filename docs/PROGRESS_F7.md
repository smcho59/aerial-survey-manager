# 작업 완료 기록 — F7

## F7. 대시보드/처리화면 베이스맵 on/off 토글 - 2026-03-13 (커밋: d81d6f5)

### 요구사항
대시보드(FootprintMap)와 처리화면(ProjectMap)에 표출되는 베이스맵 레이어를 껐다 켤 수 있게 한다.
정사영상 위에 베이스맵이 불필요하거나 방해되는 경우 끌 수 있다.

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
- [x] D4. 커밋 (d81d6f5)

### A1. 구현 계획

#### 기존 코드 현황
- **FootprintMap.jsx**: TileLayer 고정 렌더링. FootprintMapHeader에 "촬영 영역"/"권역" 토글 버튼 존재
- **ProjectMap.jsx**: TileLayer 고정 렌더링. 레이어 컨트롤 UI 없음
- **mapConfig.js**: `getTileConfig()`로 오프라인/온라인 설정 반환

#### 변경 파일 (2개 FE + 1개 정적자산 + 1개 설정)

**1. `src/components/Dashboard/FootprintMap.jsx`**
- `showBasemap` 상태 추가 (localStorage `basemap_visible` 키 공유)
- 지도 위 플로팅 토글 버튼 추가 (우상단, MapIcon 40px)
- TileLayer를 `showBasemap` 조건부 렌더링
- `Map` → `Map as MapIcon` alias (JS 내장 Map 충돌 해결)

**2. `src/components/Project/ProjectMap.jsx`**
- 동일한 `showBasemap` 상태 (같은 localStorage 키)
- 동일한 플로팅 토글 버튼 (우상단, MapIcon 40px)
- TileLayer를 `showBasemap` 조건부 렌더링

**3. `public/siqms_mark.png`** — 로고 복구
- 이전 프로덕션 Docker 이미지(`aerial-prod-frontend:latest`)에서 추출
- Vite `public/` 디렉토리에 배치하여 빌드 시 자동 포함

**4. `.gitignore`** — `!public/**/*.png` 예외 추가

### A2. 계획 검토 결과
- 조건부 렌더링이 opacity 0보다 나음: 불필요한 타일 요청 방지
- localStorage 동기화: 같은 키를 쓰므로 대시보드에서 끄면 처리화면 진입 시에도 꺼진 상태
- 백엔드 변경 없음 — 순수 프론트엔드 작업

### A3. 과도 설계 검토 결과
- 별도 공통 컴포넌트 분리 불필요: 두 곳의 UI 배치가 동일(플로팅 버튼)이나 각각 인라인으로 충분
- React Context 불필요: 두 맵이 동시 렌더링되지 않으므로 localStorage로 충분

### 변경 파일 요약

| 파일 | 변경 내용 |
|---|---|
| `src/components/Dashboard/FootprintMap.jsx` | showBasemap 상태, 플로팅 토글 버튼, Map→MapIcon alias |
| `src/components/Project/ProjectMap.jsx` | showBasemap 상태, 플로팅 토글 버튼 |
| `public/siqms_mark.png` | 로고 이미지 복구 (이전 Docker 이미지에서 추출) |
| `.gitignore` | `!public/**/*.png` 예외 추가 |

### 회귀 기록
(없음)

### 발견된 이슈 및 결정사항
- `Map` import가 JS 내장 `Map` 객체와 충돌하여 흰 화면 발생 → `Map as MapIcon` alias로 해결
- `siqms_mark.png`가 git에 없었음 (`.gitignore`의 `*.png` 규칙) → `!public/**/*.png` 예외 추가
- 로고 파일은 이전 프로덕션 Docker 이미지(`aerial-prod-frontend:latest`)에서 복구
