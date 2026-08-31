# Team_E2 — 숨은 물체 탐색 협동로봇 어시스턴트

AI(Computer Vision) 기반 협동로봇 작업 어시스턴트 구현 프로젝트입니다.
사용자가 음성으로 물체를 요청하면, 로봇이 카메라로 주변을 관측하고 —
바로 보이지 않으면 서랍·문을 열어 내부를 재관측하는 탐색 행동을 반복해 —
대상을 찾아 파지한 뒤 지정 위치로 옮깁니다. 전체 과정(현재 판단, 탐색 단계,
물체 위치, 작업 기록)은 웹 대시보드 형태의 관측 전용 UI로 실시간 확인할 수
있습니다.

## 핵심 동작 흐름

1. 음성 명령 수신 (STT + LLM으로 물체명 보정/분류)
2. DB에 마지막 위치가 있으면 그 위치부터 우선 관측
3. 대상이 안 보이면 사전 정의된 탐색 구역을 순서대로 이동
4. 서랍/기프트박스(green_box·gray_box) 열기
5. 내부 재관측
6. 물체 후보 확인 (Any6D 6D pose 검증)
7. 파지 접근 및 파지 검증 (MoveIt 2 + RG2 그리퍼)
8. 지정 위치로 이송 및 사용자 전달, 열었던 보관함 원상복구
9. 홈 자세 복귀, 성공/실패/취소 기록

로봇이 단순히 화면 속 물체를 검출하는 것이 아니라, 대상이 안 보일 때
서랍·문을 열고 재관측하며 탐색 행동을 반복하는 것이 이 프로젝트의 핵심입니다.

---

## 시스템 아키텍처

```
음성(voice_command) ──TargetSearch──▶ 상태머신(state) ──Search action──▶ 제어(control_node)
                                          │  ▲                              │  ▲
                                          │  │RobotResult                   │  │DetectObject
                                     DbSave│  │                    DbLoad(위치)│  │UpdateTcpPose
                                          ▼  │                              ▼  │
                                        DB(db) ◀────────DbSave(items)──── 비전(vision_nodes)
                                          │                                   │
                                     DbLoad(폴링)                    MoveIt2 / RG2 그리퍼 / Doosan M0609
                                          ▼
                                    UI 어댑터(back_ui) ──HTTP(JSON)──▶ 대시보드(front_ui)
                                          ▲
                                  /ui/task_state, /state/current
                                  (state가 발행)
```

| 패키지 | 역할 |
|---|---|
| `interfaces` | 전체 노드가 공유하는 ROS 2 srv/action 정의 |
| `db` | SQLite 기반 물체 위치(`items`)/작업 기록(`tasks`) 저장·조회 서비스 |
| `state` | 전체 상태머신(LOAD→IDLE→RUN). 명령 접수, 탐색 액션 발행, 제어 노드 진행 보고 중계, 작업 기록 저장 |
| `control_node` | MoveIt 2 기반 로봇 제어. DB 위치 우선 확인 → 구역 탐색 → Any6D 파지 검증 → RG2 파지 → 이송·전달·보관함 복구까지 전체 작업 관리 |
| `back_ui` | ROS 토픽/서비스를 front_ui가 쓰는 HTTP(JSON) 계약으로 변환하는 어댑터 |
| `front_ui` | Flet 기반 관측 전용 대시보드(홈/작업/로그 3개 화면 + 3D 지도) |
| `voice_command` | 웨이크워드("Hello Rokey") 감지 + STT(Whisper) + LLM 기반 물체명 분류 → 상태머신에 명령 전달 |
| `vision_nodes` | GroundingDINO + Any6D 기반 객체 탐지/6D pose 추정, DB 저장 요청 |

---

## 기술 스택

| 분류 | 사용 기술 |
|---|---|
| 로봇 미들웨어 | ROS 2 Humble (rclpy) |
| 로봇 | Doosan M0609 (`doosan-robot2` 드라이버), MoveIt 2 |
| 그리퍼 | OnRobot RG2 (PyModbus로 Tool Changer 제어) |
| 상태/서비스 통신 | ROS 2 service / action, JSON-in-string 페이로드 |
| 데이터베이스 | SQLite3 (화이트리스트 기반 동적 스키마 접근) |
| UI | Flet(Flutter 기반 파이썬 UI 프레임워크), HTTP 폴링 |
| 3D 시각화 | numpy 기반 자체 투영/셰이딩(외부 3D 엔진 미사용) |
| 음성 인식 | OpenAI Whisper API(STT), GPT(물체명 정규화), openWakeWord(웨이크워드) |
| 비전 | GroundingDINO(오픈보캐뷸러리 검출), Any6D(6D pose 추정) |
| 카메라 | Intel RealSense (RGB-D, Eye-in-Hand) |

---

## 저장소 구조

```
Team_E2/
├── interfaces/       # 공용 srv/action 정의 (rosidl)
├── db/               # DB 노드 (SQLite items/tasks)
├── state/            # 상태머신 노드 + 통합 launch 파일
├── control_node/     # MoveIt 2 기반 로봇 제어 노드
├── back_ui/          # ROS ↔ HTTP 어댑터 노드
├── front_ui/         # Flet 대시보드 (독립 conda 환경)
│   ├── src/          # 앱 소스 (views, render3d, client, components)
│   ├── tools/        # 개발/데모용 스크립트 (fake_server 등)
│   └── tests/        # pytest 테스트
├── voice_command/    # 음성 명령 노드
└── vision_nodes/     # 비전 노드 (GroundingDINO/Any6D)
```

각 패키지 디렉터리 안의 `README.md`에 더 자세한 내용이 있습니다.

---

## 사전 준비물

| 항목 | 버전/비고 |
|---|---|
| OS | Ubuntu 22.04 |
| ROS 2 | Humble Hawksbill |
| Python | ROS 2 시스템 파이썬(3.10) + front_ui 전용 conda 환경(3.11) |
| Miniconda/Anaconda | front_ui, 비전 파이프라인 각각 독립 환경으로 실행 |
| Doosan M0609 로봇 드라이버 + MoveIt 2 설정 | 같은 워크스페이스의 `doosan-robot2` 저장소(별도) — 실물 로봇 또는 시뮬레이션용 |
| OnRobot RG2 그리퍼 | Tool Changer 또는 RG2 Modbus 연결 장치(기본 `192.168.1.1:502`) |
| OpenAI API 키 | `voice_command`의 STT/LLM 호출용 |
| Intel RealSense 카메라 | `back_ui` 카메라 스트림, `vision_nodes` 객체 검출·`control_node` Any6D 파지용 (선택 — 없어도 UI 화면은 정상 동작) |
| GroundingDINO / Any6D 환경 | `vision_nodes` 전용 conda 환경 — 자세한 설치는 `vision_nodes/README.md` 참고 |

---

## 설치

### 1. 워크스페이스 준비

```bash
mkdir -p ~/cobot_ws/src
cd ~/cobot_ws/src
git clone <이 저장소 URL> Team_E2
```

`doosan-robot2`(로봇 드라이버) 등 이 저장소가 의존하는 다른 패키지들도
같은 `~/cobot_ws/src` 아래에 함께 있어야 합니다.

### 2. ROS 2 의존성 설치

```bash
cd ~/cobot_ws
source /opt/ros/humble/setup.bash
rosdep update
rosdep install --from-paths src --ignore-src -r -y
```

### 3. 빌드 (interfaces를 먼저 빌드)

`interfaces`는 다른 모든 패키지가 의존하는 공용 메시지 패키지이므로 먼저
단독으로 빌드한 뒤 전체를 빌드합니다.

```bash
colcon build --packages-select interfaces
source install/setup.bash
colcon build --symlink-install
source install/setup.bash
```

> `~/cobot_ws/src` 안에 `control_node`라는 이름의 ROS 2 패키지가 이 저장소 것
> 말고 또 있으면(예: 별도로 풀어둔 개발용 사본) colcon이 "Duplicate package
> names" 오류를 냅니다. 그 폴더에 빈 `COLCON_IGNORE` 파일을 만들어 빌드
> 대상에서 빼거나, 둘 중 쓸 버전 하나만 남기세요.

### 4. front_ui 전용 conda 환경 생성

front_ui는 ROS와 완전히 분리된 별도 프로세스로 동작합니다(`rclpy`를
가져오지 않습니다).

```bash
cd ~/cobot_ws/src/Team_E2/front_ui
conda env create -f environment.yaml
conda activate front_ui
```

### 5. voice_command 환경 변수 설정

`voice_command/resource/.env` 파일을 만들고 OpenAI API 키를 넣습니다.

```bash
echo "OPENAI_API_KEY=sk-..." > ~/cobot_ws/src/Team_E2/voice_command/resource/.env
```

`voice_command`는 별도 conda 환경 없이 ROS 2가 쓰는 시스템 파이썬에서
동작하므로, 아래 패키지들을 해당 파이썬에 설치합니다.

```bash
pip install openai python-dotenv sounddevice scipy numpy pyaudio openwakeword
```

### 6. (선택) 비전 파이프라인 환경

`vision_nodes`는 GroundingDINO/Any6D/PyTorch 등 별도 conda 환경을
전제로 합니다. 자세한 설치 절차는 `vision_nodes/README.md`를 참고하세요.

### 7. (실물 로봇 사용 시) 로봇/그리퍼 준비

- `doosan-robot2`의 M0609 드라이버 + MoveIt 2 설정 패키지가 빌드되어 있어야 합니다.
- OnRobot RG2가 `192.168.1.1:502`(기본값)로 연결 가능해야 합니다.
- Eye-in-Hand 카메라의 `TCP -> Camera` 외부 파라미터 보정이 끝나 있어야 합니다.

자세한 요구사항은 `control_node/README.md` 1장을 참고하세요.

---

## 실행 방법

### A. UI 단독 데모 모드 (로봇/DB 없이 화면만 확인)

실제 하드웨어나 ROS 파이프라인 없이 front_ui 화면만 확인하고 싶을 때
씁니다. `front_ui/tools/fake_server.py`가 `back_ui`와 동일한 HTTP 계약으로
가상의 상태값을 계속 만들어 보냅니다.

```bash
# 터미널 1
conda activate front_ui
cd ~/cobot_ws/src/Team_E2/front_ui
python tools/fake_server.py

# 터미널 2
conda activate front_ui
cd ~/cobot_ws/src/Team_E2/front_ui
flet run
```

### B. 전체 파이프라인 (실물/시뮬레이션 로봇 포함)

노드마다 서로를 기다리는 지점이 있어서, 아래 순서로 띄우는 것을 권장합니다.
전부 `source ~/cobot_ws/install/setup.bash`를 먼저 실행한 상태여야 합니다.

```bash
# 1) 로봇 드라이버 + MoveIt 2 (doosan-robot2 저장소, 같은 워크스페이스)
ros2 launch dsr_moveit_config_m0609 start_2.launch.py \
  mode:=real model:=m0609 name:=dsr01 host:=<로봇 IP> gui:=true

# 2) DB
ros2 run db db_node

# 3) 제어 노드 — 1)의 move_group/joint_states/TF가 떠 있어야 정상 기동
ros2 launch control_node control_node.launch.py

# 4) 비전 (별도 conda 환경, GPU 필요)
conda activate <vision 환경>
ros2 launch vision_nodes vision_nodes.launch.py

# 5) HTTP 어댑터
ros2 run back_ui node

# 6) 상태머신 — targets=['db','control'], search_action=/control/search가 기본값
ros2 run state state_node

# 7) 음성 명령
ros2 run voice_command voice_command_node

# 8) front_ui (conda 환경)
conda activate front_ui
cd ~/cobot_ws/src/Team_E2/front_ui
flet run
```

2·5·6은 `state/launch/team_e2.launch.py` 하나로 묶어서 띄울 수 있습니다(아래 D 참고).
1·3·4는 하드웨어/GPU conda 환경에 따라 별도 터미널로 남겨둡니다.

**부팅 순서가 중요한 이유**: `state_node`는 `targets`에 지정된 각 노드의
`init` 서비스가 응답할 때까지 재시도하며 기다리므로 순서가 엄격하게
강제되지는 않지만, `control_node`는 시작 시점에 `/move_action`,
`/execute_trajectory`, `/joint_states` 등 MoveIt 2 쪽이 이미 떠 있을
것을 전제로 하므로 로봇 드라이버 → MoveIt 2 → 제어 노드 순서를 지키는
것이 안전합니다.

### C. 로봇 없이 상태머신·UI만 확인

`control_node`(및 로봇)를 아직 준비하지 못했다면 `state_node`의
`targets` 파라미터에서 `control`을 빼고 띄우면 `db`만 확인하고 바로
`IDLE`로 넘어갑니다. 이 경우 실제 탐색 액션은 아무도 응답하지 않으므로
음성/수동 명령은 "탐색 노드 없음"으로 거절됩니다 — DB/UI 연동만 확인할
때 씁니다.

```bash
ros2 run state state_node --ros-args -p targets:="['db']"
```

### D. 통합 launch (db + back_ui + state 한 번에)

```bash
ros2 launch state team_e2.launch.py
```

---

## ROS 2 인터페이스 요약

| 이름 | 타입 | 제공 노드 | 용도 |
|---|---|---|---|
| `NodeInit` | srv | 전 노드 공용 | 상태머신의 기동 확인 요청 |
| `DbSave` | srv | `db` | 물체(items)/작업기록(tasks) 저장 |
| `DbLoad` | srv | `db` | 물체/작업기록 조회 |
| `TargetSearch` | srv | `state` | 음성 명령 → 상태머신 접수 |
| `RobotResult` | srv | `state` | 제어 노드의 작업 진행 보고 수신 |
| `ControlTask` | srv | `control_node` | 상태머신/수동 호출 → 제어 작업 지시 |
| `DetectObject` | srv | `vision_nodes` | 제어 → 비전 객체 탐지 요청(`/find_object_pose`) |
| `UpdateTcpPose` | srv | `vision_nodes` | 제어 → 비전에 현재 TCP 자세 전달 |
| `Search` | action | `control_node` | 상태머신 → 제어 탐색·파지 실행 |

각 서비스의 정확한 JSON 필드는 해당 패키지의 README(`db`, `state`,
`control_node`)를 참고하세요.

---

## 데이터베이스 스키마

SQLite, 기본 경로 `~/.ros/robot_db/robot.db`.

**`items`** — 마지막으로 확인된 물체 위치 (class_name 기준 upsert)

| 컬럼 | 타입 | 비고 |
|---|---|---|
| class_name | TEXT UNIQUE | 비전 노드의 검출 클래스명 |
| confidence | REAL | 검출 신뢰도 |
| x, y, z | REAL | 로봇 base 기준 좌표(mm) |
| last_seen | TEXT | ISO8601 |

**`tasks`** — 작업 완료 시점 1회 기록

| 컬럼 | 타입 | 비고 |
|---|---|---|
| command_text | TEXT | 예약 필드 |
| voice_command | TEXT | 사용자가 말한 원문 음성 텍스트 |
| target_name | TEXT | 대상 물체 |
| destination | TEXT | 예약 필드 |
| status | TEXT | SUCCEEDED / FAILED / ABORTED |
| fail_stage | TEXT | 실패 시 단계 |
| fail_reason | TEXT | 실패 사유 |
| found_at | TEXT | 발견 위치 |
| started_at / ended_at | TEXT | ISO8601 |

지원 물체(등록된 6D 모델): `yellow_can`, `green_box`, `gray_box`,
`white_bear`, `aircon_remote`, `green_frog`, `otter_in_can`.

---

## front_ui 개발 도구 (`front_ui/tools/`)

| 스크립트 | 용도 |
|---|---|
| `fake_server.py` | `back_ui`와 동일한 HTTP 계약(`/state`, `/health`, `/frame.jpg`)을 흉내내는 가짜 서버. 로봇/DB 없이도 UI 화면 데이터를 확인할 수 있게 해준다 |
| `mesh_to_points.py` | 로봇 팔 COLLADA mesh → 3D 지도용 점군(.npy) 오프라인 변환 |
| `render_object.py` | 물체 OBJ → 정적 PNG 오프라인 렌더 (뷰어 폴백용) |
