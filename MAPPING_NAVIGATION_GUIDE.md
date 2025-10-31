# Go2 로봇 매핑 및 네비게이션 가이드

Go2 ROS2 SDK에서 맵 생성과 자율 네비게이션을 사용하는 방법을 설명합니다.

---

## 목차

1. [시스템 개요](#시스템-개요)
2. [사전 준비](#사전-준비)
3. [1단계: 맵 생성 (Mapping)](#1단계-맵-생성-mapping)
4. [2단계: 맵 저장](#2단계-맵-저장)
5. [3단계: 네비게이션 설정](#3단계-네비게이션-설정)
6. [4단계: 자율 네비게이션 실행](#4단계-자율-네비게이션-실행)
7. [문제 해결](#문제-해결)

---

## 시스템 개요

### 사용되는 기술 스택

| 구성 요소 | 역할 | 사용 시기 |
|----------|------|----------|
| **SLAM Toolbox** | 실시간 맵 생성 (mapping 모드) | 맵 생성 단계 |
| **AMCL** | 파티클 필터 기반 위치 추정 | 네비게이션 단계 |
| **Nav2** | 경로 계획 및 장애물 회피 | 네비게이션 단계 |
| **Map Server** | 저장된 맵 로드 및 퍼블리시 | 네비게이션 단계 |

### 맵 형식

- **실시간**: `nav_msgs/OccupancyGrid` (ROS2 토픽)
- **저장**: YAML + PGM 파일
  - `map.yaml`: 메타데이터 (해상도, 원점 등)
  - `map.pgm`: 점유 격자 이미지 (0-255 그레이스케일)

---

## 사전 준비

### 1. 로봇 연결 설정

환경 변수 설정:
```bash
export ROBOT_IP="192.168.1.100"      # 로봇 IP 주소
export ROBOT_TOKEN="your_token"      # 로봇 인증 토큰
export CONN_TYPE="webrtc"            # webrtc 또는 cyclonedds
```

### 2. 필수 패키지 확인

```bash
# ROS2 워크스페이스에서
cd ~/go2_ros2_sdk
source install/setup.bash

# 패키지 빌드 확인
ros2 pkg list | grep -E "slam_toolbox|nav2|go2_robot_sdk"
```

---

## 1단계: 맵 생성 (Mapping)

### 실행 명령

```bash
ros2 launch go2_robot_sdk robot.launch.py \
  slam:=true \
  nav2:=false \
  rviz2:=true
```

**파라미터 설명**:
- `slam:=true`: SLAM Toolbox 활성화 (맵 생성)
- `nav2:=false`: Nav2 비활성화 (TF 충돌 방지)
- `rviz2:=true`: RViz 시각화 활성화

### RViz에서 확인

실행 후 RViz에서 다음 항목을 확인:

1. **PointCloud2** (활성화됨)
   - 토픽: `/point_cloud2`
   - 3D LiDAR 포인트 클라우드
   - 색상: Intensity 기반 무지개색

2. **Map** (비활성화 상태 → 수동으로 활성화 필요)
   - 토픽: `/map`
   - 2D 점유 격자 맵
   - 색상: 흑백 (검정=장애물, 흰색=빈 공간, 회색=미지)

**Map 디스플레이 활성화 방법**:
- RViz 좌측 패널에서 `Map` 항목 찾기
- 체크박스 클릭하여 활성화

### 맵 생성 과정

1. **로봇 이동**
   - 조이스틱 또는 텔레오퍼레이션으로 로봇 조작
   - 환경을 천천히 탐색하며 이동
   - 모든 영역을 골고루 방문

2. **실시간 맵 확인**
   - RViz에서 `/map` 토픽의 점유 격자 맵 확인
   - 검정색 영역: 벽, 장애물
   - 흰색 영역: 이동 가능한 빈 공간
   - 회색 영역: 아직 탐색하지 않은 미지 영역

3. **루프 클로징**
   - 이전에 방문한 장소로 돌아가면 자동으로 루프 감지
   - 누적된 오도메트리 오차 자동 보정
   - 맵 품질 향상

### SLAM 설정 (고급)

설정 파일: `go2_robot_sdk/config/mapper_params_online_async.yaml`

```yaml
slam_toolbox:
  mode: mapping                     # 맵 생성 모드
  resolution: 0.05                  # 맵 해상도: 5cm
  max_laser_range: 20.0             # 최대 LiDAR 거리: 20m
  do_loop_closing: true             # 루프 클로징 활성화
  loop_search_maximum_distance: 3.0 # 루프 탐색 거리: 3m
```

---

## 2단계: 맵 저장

맵 생성이 완료되면 저장합니다.

### 방법 1: RViz SLAM Toolbox 플러그인 (권장)

1. RViz 좌측 하단에서 **SlamToolboxPlugin** 패널 찾기
2. **"Serialize Map"** 버튼 클릭
   - 기본 위치: `~/.ros/`
   - 파일: `slam_toolbox_map.posegraph`, `slam_toolbox_map.data`

3. **"Save Map"** 버튼 클릭
   - 점유 격자 맵 저장
   - 생성 파일: `map.yaml`, `map.pgm`

### 방법 2: 명령줄

```bash
# 새 터미널 열기
source ~/go2_ros2_sdk/install/setup.bash

# 맵 저장 (현재 디렉토리에 저장)
ros2 run nav2_map_server map_saver_cli -f my_robot_map

# 또는 전체 경로 지정
ros2 run nav2_map_server map_saver_cli -f ~/maps/office_map
```

**생성되는 파일**:
```
my_robot_map.yaml    # 맵 메타데이터
my_robot_map.pgm     # 점유 격자 이미지
```

### 맵 파일 확인

**my_robot_map.yaml 예시**:
```yaml
image: my_robot_map.pgm
resolution: 0.050000
origin: [-10.000000, -10.000000, 0.000000]
negate: 0
occupied_thresh: 0.65
free_thresh: 0.25
mode: trinary
```

**my_robot_map.pgm**:
- 그레이스케일 이미지
- 이미지 뷰어로 열어서 확인 가능

---

## 3단계: 네비게이션 설정

### Nav2 설정 파일 수정

파일: `go2_robot_sdk/config/nav2_params.yaml`

**Line 269 수정**:
```yaml
map_server:
  ros__parameters:
    use_sim_time: False
    yaml_filename: "/home/user/maps/my_robot_map.yaml"  # 절대 경로 사용
```

**중요**:
- 반드시 **절대 경로**를 사용하세요
- 상대 경로는 작동하지 않을 수 있습니다
- 파일 존재 여부 확인: `ls -la /home/user/maps/my_robot_map.yaml`

### 설정 확인

다음 항목들이 올바르게 설정되어 있는지 확인:

**AMCL 설정** (Line 14):
```yaml
amcl:
  global_frame_id: "map"  # ✅ "map"이어야 함 ("odom" 아님)
```

**BT Navigator** (Line 45):
```yaml
bt_navigator:
  global_frame: map  # ✅ map
```

**Global Costmap** (Line 226):
```yaml
global_costmap:
  global_frame: map  # ✅ map
```

**Local Costmap** (Line 186):
```yaml
local_costmap:
  global_frame: odom  # ✅ odom (rolling window)
```

---

## 4단계: 자율 네비게이션 실행

### 실행 명령

```bash
ros2 launch go2_robot_sdk robot.launch.py \
  slam:=false \
  nav2:=true \
  rviz2:=true
```

**파라미터 설명**:
- `slam:=false`: SLAM Toolbox 비활성화
- `nav2:=true`: Nav2 활성화 (AMCL + 경로 계획)
- `rviz2:=true`: RViz 시각화

### 시스템 구성

```
Map Server (저장된 맵 로드)
    ↓
AMCL (위치 추정)
    ├─ 파티클 필터로 로봇 위치 추정
    └─ TF: map → odom 퍼블리시
    ↓
Nav2 Navigation Stack
    ├─ Global Planner: SMAC Planner Hybrid
    ├─ Local Planner: DWB (Dynamic Window Approach)
    └─ Costmap: 장애물 감지 및 회피
```

### RViz에서 네비게이션 사용

1. **초기 위치 설정 (Initial Pose)**
   - RViz 상단 도구 모음에서 **"2D Pose Estimate"** 버튼 클릭
   - 맵에서 로봇의 현재 위치를 클릭하고 드래그하여 방향 지정
   - AMCL 파티클들이 해당 위치 주변에 분포

2. **목표 지점 설정 (Goal)**
   - RViz 상단 도구 모음에서 **"Nav2 Goal"** 또는 **"2D Goal Pose"** 버튼 클릭
   - 목표 위치를 클릭하고 드래그하여 도착 방향 지정
   - 자동으로 경로 계획 후 이동 시작

3. **시각화 확인**
   - **Global Plan** (녹색 선): 전역 경로
   - **Local Plan** (빨간색 선): 로컬 경로 (장애물 회피)
   - **Costmap**: 장애물 주변의 비용 지도
   - **Particles** (빨간색 화살표): AMCL 파티클 분포

### 명령줄에서 목표 전송

```bash
# 새 터미널
source ~/go2_ros2_sdk/install/setup.bash

# 목표 위치 전송 (x=2.0, y=1.0, yaw=0.0)
ros2 topic pub --once /goal_pose geometry_msgs/PoseStamped \
'{
  header: {frame_id: "map"},
  pose: {
    position: {x: 2.0, y: 1.0, z: 0.0},
    orientation: {x: 0.0, y: 0.0, z: 0.0, w: 1.0}
  }
}'
```

---

## 문제 해결

### 1. AMCL이 위치를 찾지 못함

**증상**: 파티클이 맵 전체에 흩어져 있음

**해결**:
- RViz에서 "2D Pose Estimate"로 초기 위치를 정확히 지정
- 로봇을 천천히 이동시켜 파티클 수렴 유도
- AMCL 파라미터 조정:
  ```yaml
  amcl:
    max_particles: 2000  # 파티클 수 증가
    min_particles: 500
  ```

### 2. 맵이 로드되지 않음

**증상**: RViz에서 맵이 보이지 않음

**확인 사항**:
```bash
# Map Server가 실행 중인지 확인
ros2 node list | grep map_server

# /map 토픽이 퍼블리시되는지 확인
ros2 topic list | grep /map
ros2 topic echo /map --once

# 맵 파일 경로 확인
cat ~/go2_ros2_sdk/go2_robot_sdk/config/nav2_params.yaml | grep yaml_filename

# 파일 존재 확인
ls -la /path/to/your/map.yaml
```

**해결**:
- `yaml_filename`에 절대 경로 사용
- 파일 권한 확인 (`chmod 644 map.yaml map.pgm`)

### 3. TF 변환 오류

**증상**: "Transform timeout" 또는 "Could not transform"

**원인**: SLAM과 AMCL이 동시에 실행되어 `map → odom` TF 충돌

**해결**:
```bash
# 네비게이션 시 반드시 slam:=false
ros2 launch go2_robot_sdk robot.launch.py slam:=false nav2:=true

# TF 트리 확인
ros2 run tf2_tools view_frames
# 생성된 frames.pdf 파일 확인
```

올바른 TF 구조:
```
map (Map Server)
  ↓ (AMCL이 퍼블리시)
odom (WebRTC 로봇 포즈)
  ↓
base_link (로봇)
```

### 4. 경로 계획 실패

**증상**: "Failed to create plan" 또는 "No valid path found"

**원인**:
- 목표 지점이 장애물 안에 있음
- 맵이 오래되어 현재 환경과 다름
- Costmap 설정 문제

**해결**:
- 목표 지점을 장애물에서 멀리 설정
- 맵을 다시 생성
- Costmap 파라미터 조정:
  ```yaml
  global_costmap:
    inflation_radius: 0.25  # 장애물 주변 안전 거리
  ```

### 5. 로봇이 회전만 하고 이동 안 함

**증상**: 제자리에서 계속 회전

**원인**: DWB 로컬 플래너 설정 문제

**해결**:
```yaml
# nav2_params.yaml
controller_server:
  FollowPath:
    min_vel_x: 0.0
    max_vel_x: 0.5  # 최대 속도 확인
    min_vel_y: 0.0
    max_vel_y: 0.5
```

### 6. 로그 확인

```bash
# Nav2 로그 확인
ros2 node list
ros2 node info /controller_server
ros2 node info /planner_server

# AMCL 상태 확인
ros2 topic echo /amcl_pose
ros2 topic echo /particlecloud
```

---

## 고급 설정

### AMCL 튜닝

파일: `go2_robot_sdk/config/nav2_params.yaml`

**위치 추정 정확도 향상**:
```yaml
amcl:
  max_particles: 5000           # 파티클 수 증가 (CPU 부하 증가)
  min_particles: 1000
  update_min_d: 0.1             # 더 자주 업데이트 (0.25 → 0.1)
  update_min_a: 0.1             # 회전 시 더 자주 업데이트
```

**넓은 환경에서 사용**:
```yaml
amcl:
  laser_max_range: 30.0         # LiDAR 최대 거리 증가
  max_beams: 120                # 사용하는 빔 수 증가
```

### Nav2 경로 계획 튜닝

**빠른 경로 계획**:
```yaml
planner_server:
  GridBased:
    tolerance: 1.0              # 목표 허용 오차 증가
    downsample_costmap: true    # 맵 다운샘플링
```

**부드러운 경로**:
```yaml
controller_server:
  FollowPath:
    max_vel_x: 0.3              # 최대 속도 감소
    max_vel_theta: 0.5          # 회전 속도 감소
```

---

## 요약

### 매핑 단계
```bash
ros2 launch go2_robot_sdk robot.launch.py slam:=true nav2:=false rviz2:=true
# 로봇 이동하며 맵 생성
ros2 run nav2_map_server map_saver_cli -f ~/maps/my_map
```

### 네비게이션 단계
```bash
# nav2_params.yaml 수정: yaml_filename 설정
ros2 launch go2_robot_sdk robot.launch.py slam:=false nav2:=true rviz2:=true
# RViz에서 "2D Pose Estimate" → "Nav2 Goal"
```

### 핵심 포인트
- ✅ 매핑 시: `slam:=true nav2:=false`
- ✅ 네비 시: `slam:=false nav2:=true`
- ✅ 둘을 동시에 실행하면 TF 충돌 발생
- ✅ 맵 파일 경로는 절대 경로 사용

---

## 참고 자료

- [SLAM Toolbox Documentation](https://github.com/SteveMacenski/slam_toolbox)
- [Nav2 Documentation](https://navigation.ros.org/)
- [AMCL Documentation](https://navigation.ros.org/configuration/packages/configuring-amcl.html)
- Go2 ROS2 SDK README: `README.md`