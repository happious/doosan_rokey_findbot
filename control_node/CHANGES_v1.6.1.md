# v1.6.1 변경 사항

- `holding_object=True`일 때 `/control/task` 요청을 거절하던 검사를 제거했습니다.
- `holding_object=True`일 때 `/control/search` 액션 목표를 거절하던 검사를
  제거했습니다.
- 이미 대기열에 들어간 작업을 실행 직전에 `blocked_holding_object`로 종료하던
  검사를 제거했습니다.
- 더 이상 사용하지 않는 `TaskOutcome.BLOCKED_HOLDING_OBJECT` 결과 코드를
  제거했습니다.
- `holding_object` 자체는 RG2 파지·해제, 사용자 전달 대기 및 상태 응답을 위해
  유지합니다.
- v1.6.0의 동적 CollisionObject 미생성 정책과 Home 자동 복구 정책은 그대로
  유지합니다.
