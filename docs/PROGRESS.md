# 작업 진행 상황

> 완료된 작업: [PROGRESS_F2.md](PROGRESS_F2.md) · [PROGRESS_F3.md](PROGRESS_F3.md) · [PROGRESS_F7.md](PROGRESS_F7.md) · [PROGRESS_F8.md](PROGRESS_F8.md)

## F6: 내보내기 후 COG 삭제 선택 - 2026-03-13
### 요구사항
- 정사영상 내보내기 완료 후, 사용자에게 COG 파일 삭제 여부를 선택할 수 있게 한다
- 저장공간 확보를 위해 COG를 삭제할 수 있되, 삭제 전 충분한 경고 제공

### 진행 체크리스트
- [x] A1. 계획 수립
- [x] A2. 계획 검토
- [x] A3. 과도 설계 검토
- [ ] 🚪 사용자 승인
- [ ] B1. 구현
- [ ] B2. 목적 부합 검토
- [ ] B3. 버그/보안 검토 및 수정
- [ ] B4. 수정사항 재검토
- [ ] C1~C5. 코드 품질 검토
- [ ] 🚪 변경사항 보고
- [ ] D1~D3. 최종 검증
- [ ] 🚪 최종 승인
- [ ] D4. 커밋

### 구현 계획

**백엔드:**
1. `DELETE /api/v1/projects/{project_id}/ortho/cog` API 추가
   - COG 파일 삭제 + `project.ortho_path = null`, `project.ortho_size = null`
   - 프로젝트 상태, bounds, area, GSD 등 메타데이터는 보존
   - 되돌릴 수 없음 (재처리 필요)

**프론트엔드:**
1. ExportDialog에서 다운로드 완료 후 COG 삭제 확인 다이얼로그 표시
   - "저장공간 절약을 위해 원본 정사영상(COG)을 삭제하시겠습니까?"
   - 파일 크기 표시, 경고 문구
   - "삭제" / "보관" 버튼
2. 삭제 성공 시 프로젝트 목록 갱신

### A2. 논리적 타당성 검토
- ExportDialog에서 다운로드 완료 시점을 감지 가능: `prepareBatchExport` → `triggerDirectDownload` 후 시점
- COG 삭제 API는 단순 파일 삭제 + DB 업데이트로 충분
- 대시보드에서 COG 삭제된 프로젝트는 TiTiler가 자연스럽게 실패하므로 정사영상 미표시

### A3. 과도 설계 검토
- SPRINT5_PLAN에는 썸네일 생성, ortho_thumbnail_path 필드, ImageOverlay 폴백 등이 있으나
  사용자 요구는 "삭제 선택권"만이므로 썸네일 관련은 제외
- DB 마이그레이션 없이 기존 ortho_path/ortho_size 필드만 null로 설정하여 구현

### 회귀 기록
(없음)

### 발견된 이슈 및 결정사항
(검토 과정에서 발견된 사항 기록)
