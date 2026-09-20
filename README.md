# SOCAMACA

TurtleBot4 기반 전투로봇 시스템 — **순찰(SLAM/Nav2) → YOLO 적 탐지 → 자동 조준 →
사람이 UI에서 발사 승인(Human-in-the-Loop) → 아두이노 슈터 발사**까지의
4개 서브시스템 통합 프로젝트입니다.

ROKEY 부트캠프 지능로봇 1조 · SOCAMACA

---

## 시스템 구성

```
  폰캠 / OAK-D
       │
       ▼
 ┌──────────────┐  /detections   ┌─────────────────┐  /goal   ┌───────────┐
 │ yolo_detector│ ─────────────▶ │ mission_manager │ ───────▶ │ SLAM/Nav2 │
 └──────────────┘                │    (상태머신)   │          └───────────┘
                                 └─────────────────┘
                                   │            │
                        /aim_error │            │ /team_status
                                   ▼            ▼
                     ┌─────────────────┐  ┌───────────────────┐
                     │ azimuth_tracker │  │  rosbridge ↔ UI   │
                     │   target_nav    │  │ (발사 승인 버튼)  │
                     └─────────────────┘  └───────────────────┘
                                                    │ /fire_cmd
                                                    ▼
                                           ┌──────────────────┐
                                           │ arduino_shooter  │
                                           └──────────────────┘
```

## 폴더 구조

```
AMR_ws/tracking/            상태머신·조준·추적 노드
                            (mission_manager, azimuth_tracker, target_nav, random_patrol)
patrol_ws/src/
  socamaca_msgs/            커스텀 ROS2 메시지 (TeamStatus, Detection, Detections)
  yolo_detector/            YOLO 탐지 노드
                            (phone_cam_detector.py=폰카메라용, bbox4.py=OAK-D용)
                            + 학습 가중치 y11s_my_best.pt 포함
socamaca-ui/
  frontend-team-pc/         팀 조작용 React UI (발사 승인 버튼 등)
  frontend-judge-pc/        심판용 React UI
  backend/                  FastAPI 백엔드 (mock 모드 지원)
  judge-vision/             심판 판정용 비전 모듈
cod/                        핑퐁볼 트래킹 실험 코드
arduino_shooter/            아두이노 슈터 ROS2 노드
judge-pc-essentials/        심판 PC 단독 구동용 최소 패키지 세트
start_all.sh / stop_all.sh  전체 파이프라인 기동/종료 스크립트
```

> `dataset/`, `.venv`, `node_modules`, colcon `build/install/log`, 학습 산출물(`runs/`)은
> 용량 문제로 이 레포에서 제외했습니다 (`.gitignore` 참고). 아래 설치 과정에서 새로 생성됩니다.
> YOLO 학습 가중치(`.pt`)는 바로 실행할 수 있도록 레포에 포함되어 있습니다.

---

## 1. 사전 준비 (시스템 레벨)

- Ubuntu 22.04 + **ROS2 Humble**
- `sudo apt install ros-humble-rosbridge-server sshpass ffmpeg`
- **TurtleBot4 표준 워크스페이스**(`turtlebot4_ws`, SLAM/Nav2 launch 파일 포함) — 이 레포에는 미포함,
  [공식 TurtleBot4 설치 가이드](https://turtlebot.github.io/turtlebot4-user-manual/)대로 별도 설치 필요
- **explore_lite**(`m-explore-ros2`) — `tracking` 패키지가 PATROL 상태에서 실행하므로
  `AMR_ws/src/`에 별도로 clone해서 같이 빌드해야 함 (원본 오픈소스라 미포함)
- Node.js 18+ / npm (프론트엔드 빌드용)
- CUDA 지원 GPU + PyTorch(CUDA) — YOLO 추론용. 없어도 CPU로 동작은 하나 훨씬 느림
- 폰 카메라를 쓰는 경우 [Iriun Webcam](https://iriun.com) 앱을 노트북·폰 양쪽에 설치

## 2. ROS2 워크스페이스 빌드

```bash
git clone https://github.com/onlyho12-sketch/SOCAMACA.git
cd SOCAMACA

# 메시지 + YOLO 탐지 노드
cd patrol_ws
colcon build --symlink-install
source install/setup.bash

# 상태머신/조준/추적 노드 (socamaca_msgs가 install 되어 있어야 함)
cd ../AMR_ws
colcon build --symlink-install --packages-select tracking
source install/setup.bash
```

`TeamStatus.msg` 등 `socamaca_msgs`의 메시지를 수정하면 `patrol_ws`를 다시 빌드하고,
그 워크스페이스를 source한 상태에서 `AMR_ws`(tracking)도 다시 빌드해야 새 필드가 반영됩니다.

## 3. Python 패키지 설치

`yolo_detector`(phone_cam_detector.py / bbox4.py)는 시스템 파이썬에 아래를 설치:

```bash
pip install ultralytics opencv-python
```

나머지는 각자 폴더의 `requirements.txt` 사용 (가상환경 권장):

```bash
cd socamaca-ui/backend      && pip install -r requirements.txt
cd socamaca-ui/judge-vision && pip install -r requirements.txt
cd cod                      && pip install -r requirements.txt
```

## 4. 프론트엔드 설치

```bash
cd socamaca-ui/frontend-team-pc  && npm install
cd socamaca-ui/frontend-judge-pc && npm install
```

각 폴더의 `.env`에서 `VITE_ROSBRIDGE_URL`이 실제 rosbridge 주소(기본 `ws://localhost:9090`)를
가리키는지 확인하세요.

## 5. 네트워크(디스커버리 서버) 설정

로봇(라즈베리파이)과 조종 노트북이 같은 ROS2 그래프를 봐야 하므로, **양쪽 모두** 아래 환경변수를
`~/.bashrc`(또는 실행 전 export)에 설정합니다. `<ROBOT_IP>`는 로봇 파이의 IP.

```bash
export ROS_DOMAIN_ID=5
export ROS_DISCOVERY_SERVER=';;;;<ROBOT_IP>:11811;'
export ROS_SUPER_CLIENT=False
```

Discovery Server 자체는 로봇 파이 쪽 `turtlebot4.service`가 띄웁니다(수동 기동 불필요).
연결이 간헐적으로 끊기면 `sudo systemctl restart turtlebot4.service`로 재시작하면 도움이 되는
경우가 많습니다.

## 6. 실행

### 한 번에 실행 (권장)

`start_all.sh` / `stop_all.sh`는 레포 위치를 자동으로 찾지만, **로봇 접속 정보는 반드시 지정해야
합니다.** 스크립트 상단 변수를 직접 수정하거나 환경변수로 넘기세요.

```bash
export ROBOT_IP=192.168.0.42        # 로봇 파이 IP
export ROBOT_USER=ubuntu            # 기본값 ubuntu
export ROBOT_PW=turtlebot4          # TurtleBot4 기본 비밀번호

./start_all.sh   # SLAM → Nav2 → 카메라 → YOLO → mission_manager → azimuth → target
                 # → rosbridge → 프론트엔드(UI) → 아두이노 슈터(SSH) 순서로 기동
./stop_all.sh    # 전체 종료
```

그 외 덮어쓸 수 있는 변수: `ROS_SETUP`, `PATROL_WS`, `AMR_WS`, `TB4_WS`, `UI_DIR`,
`ROBOT_WS`, `ROS_NS`(기본 `/robot4`). 로그는 `/tmp/socamaca_logs/`에 저장됩니다.

### 수동으로 단계별 실행 (디버깅 시)

```bash
# 1) SLAM
ros2 launch turtlebot4_navigation slam.launch.py namespace:=robot4
# 2) Nav2
ros2 launch turtlebot4_navigation nav2.launch.py namespace:=robot4
# 3) 폰 카메라 (Iriun Webcam 앱 먼저 폰-노트북 연결)
iriunwebcam
# 4) YOLO 탐지 (폰카메라 버전)
ros2 run yolo_detector phone_cam_detector --ros-args -r __ns:=/robot4
# 5) 상태머신
ros2 run tracking mission_manager --ros-args -r __ns:=/robot4
# 6) 조준
ros2 run tracking azimuth --ros-args -r __ns:=/robot4
# 7) 추적
ros2 run tracking target --ros-args -r __ns:=/robot4
# 8) UI <-> ROS 브릿지
ros2 launch rosbridge_server rosbridge_websocket_launch.xml
# 9) 프론트엔드
cd socamaca-ui/frontend-team-pc && npm run dev
# 10) 아두이노 슈터 (로봇 파이에서, SSH로 접속 후)
ros2 run turtlebot4_beep arduino_shooter_node --ros-args -r __ns:=/robot4 -p port:=/dev/ttyACM0
```

브라우저에서 `http://localhost:5173` 접속 → BLUE 팀 로그인(`blue1234`) → 상태/카메라/승인 확인.

YOLO 가중치는 기본적으로 `patrol_ws/src/yolo_detector/y11s_my_best.pt`를 사용합니다.
다른 가중치를 쓰려면 `export YOLO_MODEL_PATH=/path/to/best.pt`로 덮어쓸 수 있습니다.

### SLAM/Nav2 없이 텔레옵으로만 테스트하는 경우

자율 순찰 없이 수동 조종 + 탐지/조준/승인/발사만 확인하려면 위 목록에서 1)·2)를 빼고
3)~10)만 실행하면 됩니다(로봇 이동은 별도 텔레옵 키보드/조이스틱 노드로).

---

## 7. 알려진 이슈

- 로봇(라즈베리파이)-노트북 간 브릿지가 간헐적으로 끊기며 `/tf`가 유실되는 경우가 있음 →
  SLAM이 스캔을 버리고(`queue is full`) 맵을 못 만들어 자동 순찰이 멈출 수 있음.
  `turtlebot4.service` 재시작 또는 라즈베리파이 재부팅으로 완화되나 근본 원인은 미해결.
- SLAM+Nav2+YOLO+rosbridge를 동시에 구동하면 와이파이 대역폭 부족으로 ping이 튈 수 있음 →
  안 쓰는 주변장치(OAK-D 카메라 등)는 뽑아두면 도움이 됨.
- **조준 오차**: `yolo_seg_detector.py`/`bbox.py` 계열에서 조준 오차를 `error_x = (u - cx) / (img_w/2)`
  형태의 픽셀 비율로 계산해 `azimuth_tracker.py`의 P제어 입력으로 그대로 사용 중. 화면 중앙 근처에서는
  실제 각도와 비슷하지만 적이 화면 가장자리에 있을수록 선형 근사 오차가 커짐 — `CameraInfo`의 `fx`,`cx`를
  이용해 `error_angle = atan2(u - cx, fx)`로 실제각을 구하는 방식으로 교체하면 개선 가능(`kp` 재튜닝 필요).
  탐지→제어 파이프라인 지연(추론+마스크처리+EMA 필터)도 오버슈트/진동의 원인이 될 수 있음.
- **탐지 스크립트 기준점 불일치**: `yolo_detector.py`(bbox 중심) / `yolo_seg_detector.py`(마스크 중심) /
  `bbox.py`(bbox 하단 75% 지점)가 서로 다른 기준점을 사용함 — 실제 로봇에 올라간 버전에 따라 조준
  기준 자체가 달라지므로, 향후 하나로 통합 필요.
- **YOLO 실시간성 저하**: `yolo_detector.py`는 `create_timer(0.03, ...)`에서 `model.predict()`를 동기
  호출해, 추론이 30ms보다 오래 걸리면 이미지 수신 콜백까지 같은 스레드에서 막혀 체감 프레임레이트가
  급락함 — `yolo_seg_detector.py`/`bbox.py`처럼 별도 추론 스레드 + 최신 프레임만 처리하는 구조로
  통일 권장. `bbox.py`에 남아있는 디버그용 `cv2.imwrite` 디스크 저장 코드도 추론 스레드를 블로킹하므로
  운영 시 제거 필요. FP16(`half()`)/TensorRT 미적용, `imgsz` 하향 등 추론 최적화 여지도 있음.
- 위 세 항목은 코드 변경 없이 원인 분석만 진행된 상태입니다.
