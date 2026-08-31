# DRL → MoveIt 2 변경 내역

## 모션 호출 매핑

| 기존 `robot_control` | `robot_control_moveit` | 변경 이유 |
|---|---|---|
| `DSR_ROBOT2.movej()` | `MoveGroup` `/move_action` | SRDF 그룹, 제한각, self/world collision을 포함해 관절 경로 계획·실행 |
| `DSR_ROBOT2.movel()` | `/compute_cartesian_path` + `/execute_trajectory` | 파지·상승·상자 조작의 직선 경로와 충돌 여부 검증 |
| `DSR_ROBOT2.ikin()` | `/compute_ik` | Planning Scene을 반영한 충돌 없는 IK 후보만 허용 |
| `get_current_posj()` | `/joint_states` | ros2_control이 제공하는 실제 관절 상태 사용 |
| `get_current_posx()` | TF `base_link` → `tcp` | MoveIt RobotModel과 같은 링크/좌표계로 Any6D 변환 |
| `task_compliance_ctrl()` 및 `get_external_torque()` | `/joint_states.effort` 변화 감시 | 제어 경로에서 DSR API 의존성을 완전히 제거 |

## 보존된 계층

다음 모듈은 패키지명 외의 통신 계약과 흐름을 유지했습니다.

- `state_interface.py`: `/control/init`, `/control/task`, `/control/search`, 작업 큐
- `db_client.py`: `/db/load` 조회와 최신 위치 선택
- `detection_client.py`: `/find_object_pose`, `/update_robot_tcp_pose`
- `camera_transform.py`, `pose_provider.py`, `pose_utils.py`: Any6D Camera→Base 변환과 파지 후보 생성
- `onrobot.py`: RG2 Modbus 연결, 폭·힘 명령, 상태 비트 검사

## v1.6.0 Planning Scene 정책

1. 제어 노드는 `/apply_planning_scene` 클라이언트를 만들지 않습니다.
2. 노드 시작 시 고정 장애물을 추가하지 않습니다.
3. 일반 요청 물체, green box 뚜껑, gray box 서랍을 world에 추가하거나 TCP에
   attach하지 않습니다.
4. 물체 해제 시 detach 또는 world 재등록도 수행하지 않습니다.
5. MoveIt의 로봇 자체 충돌 검사와 외부 노드가 등록한 환경 충돌 검사는 유지됩니다.

따라서 파지물 형상 때문에 가상 바닥과 충돌하여 경로가 차단되는 현상은 제거되지만,
MoveIt은 운반 중인 실제 물체의 부피를 고려하지 않습니다.

## 실패 처리

`MoveItPlanningError`, `MoveItIKError`, `MoveItExecutionError`는 기존 작업 실패
보고 경로로 전달됩니다. 오류 발생 후에는 먼저 충돌회피 홈 계획을 시도하고,
복구 결과를 `/state/robot_result`의 `extra.recovery_success`와 `extra.recovery`에
기록합니다. v1.6.0에서는 Planning Scene 부착 상태와 `holding_object`의 불일치
검사를 수행하지 않으며, 오류 발생 시 항상 Home 계획을 시도합니다.

v1.6.1부터는 `holding_object=True`도 새 `/control/task` 요청, `/control/search`
액션 목표 또는 이미 대기 중인 작업의 실행을 차단하지 않습니다. 이 값은 RG2의
파지·해제와 사용자 전달 상태를 관리하고 상태 응답에 표시하는 용도로만 유지됩니다.

## 현장 적용 전 검사

1. `ros2 action list`에 `/move_action`, `/execute_trajectory` 확인
2. `ros2 service list`에 `/compute_ik`, `/compute_cartesian_path` 확인
3. `ros2 topic echo /joint_states --once`에서 6개 joint 이름과 effort 확인
4. `ros2 run tf2_ros tf2_echo base_link tcp`가 실제 RG2 TCP와 일치하는지 확인
5. RViz PlanningScene에 이 패키지가 생성한 고정·동적 물체가 없는지 확인
6. 실제 구동 전 낮은 velocity/acceleration scaling으로 계획만 검증
