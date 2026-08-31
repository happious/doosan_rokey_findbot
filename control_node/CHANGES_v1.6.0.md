# v1.6.0 변경 사항

- 일반 파지물, `green_box` 뚜껑, `gray_box` 서랍의 동적
  `CollisionObject`/`AttachedCollisionObject` 생성 로직을 제거했습니다.
- `/apply_planning_scene` 클라이언트와 World Add, Attach, Detach, 재등록 호출을
  모두 제거했습니다.
- `PlanningSceneConfig`, 파지물 크기 설정, `touch_links`,
  `attached_object_id` 추적 상태를 제거했습니다.
- Planning Scene 부착 상태와 `holding_object`가 다를 때 자동 Home 복구를
  차단하던 검사를 제거했습니다. 오류 발생 시 Home 계획을 바로 시도합니다.
- MoveIt의 로봇 자체 충돌 검사와 외부에서 등록된 환경 충돌 검사는 유지됩니다.
- 동적 파지물 형상이 더 이상 Planning Scene에 들어가지 않으므로, 실제 운반물과
  바닥·설비 사이의 안전 여유는 목표 Pose와 경로에서 확보해야 합니다.
- v1.5.6이 남긴 기존 부착물은 v1.6.0이 삭제하지 않으므로, 최초 전환 시
  `control_node`와 `move_group`을 함께 재시작해 Scene을 초기화해야 합니다.
