# control_node v1.5.4 변경 내역

- 노드 시작 시 MoveIt Planning Scene에 `work_table`과 `shelf`를 자동 등록하던
  `add_static_scene()` 호출과 구현을 삭제했습니다.
- 테이블·선반의 예시 크기 및 위치 설정을 삭제했습니다.
- 일반 요청 물체, `green_box` 뚜껑, `gray_box` 서랍의 Attach/Detach는 파지 중
  로봇과 물체의 충돌 검사를 위해 유지했습니다.
- v1.5.3의 green_box 전달 후 2초 지연 복구 흐름은 그대로 유지했습니다.
- 실제 M0609+RG2 TF에 맞춰 기본 말단 링크를 `rg2_tcp`로 정리했습니다.
- Any6D 카메라 Pose와 Base 변환 결과 XYZ를 남기는 진단 로그를 추가했습니다.
- 테스트의 이전 모듈명 `robot_control_moveit` 참조를 실제 패키지명
  `control_node`로 통일했습니다.
