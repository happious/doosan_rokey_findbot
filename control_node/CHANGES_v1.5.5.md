# v1.5.5 변경 사항

- `green_box` 뚜껑 파지점의 물체 로컬 Z 오프셋을 `+15 mm`에서 `+30 mm`로
  변경했습니다.
- `green_box` 랜드마크에 한해 카메라→Base 변환 후 로컬 오프셋을 회전까지
  반영해 적용하고, 최종 파지점 Z로 안전 하한을 검사합니다.
- 변환 로그에 raw CAD 원점 Z와 오프셋 적용 후 `Object Z`를 함께 표시합니다.
- `TargetPose.object_matrix`에는 raw CAD 원점을 유지하므로 실제 뚜껑 열기
  모션에서 오프셋은 기존과 동일하게 한 번만 적용됩니다.
- v1.5.4의 `rg2_tcp`, 정적 장애물 미생성, 파지물 Attach/Detach 및 사용자
  전달 후 green_box 지연 복구 흐름을 유지합니다.
