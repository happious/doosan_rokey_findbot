# control_node v1.5.1

- `green_box` CAD 원점 기준 파지 오프셋을 물체 로컬 `(0, 0, +15) mm`로 변경했습니다.
- `yellow_can` 등 DB 좌표가 있는 물체가 `search_zone=0` 경로를 사용하던 분기를 제거했습니다.
- 모든 물체가 동일하게 설정된 탐색구역 1~6을 순서대로 사용합니다.
- DB 조회 키를 실제 `items.class_name` 스키마에 맞췄습니다.
- DB의 Base 기준 `x/y/z`는 상태 메타데이터로 유지하되 모션 경로를 바꾸지 않습니다.
- v1.5 압축본의 실행 진입점, launch 패키지명, ament resource 이름을 `control_node`로 통일했습니다.
