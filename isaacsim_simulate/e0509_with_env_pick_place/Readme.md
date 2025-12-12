# ROS2 토픽 관계도 (Gripper 포함)

## 전체 구조

```
┌─────────────────────────────────────────────────────────────────────┐
│                   ROS2 토픽 흐름도 (with Gripper)                     │
└─────────────────────────────────────────────────────────────────────┘

┌──────────────────────────┐
│ gui_add_cartesian_       │  (PyQt5 GUI 애플리케이션)
│ control.py               │
│  - 탭 기반 제어 ✅       │
│  - Joint Control 탭      │
│  - Cartesian Control 탭  │
│  - 포즈 편집/저장        │
│  - 시퀀스 실행           │
│  - 그리퍼 제어 ✅        │
└──────────────────────────┘
         │
         ├─ 구독: /joint_states (sensor_msgs/JointState)
         │       ↳ MoveIt2 로봇 상태 (6개 관절)
         │
         ├─ 구독: /joint_pos (std_msgs/Float32MultiArray) ✅
         │       ↳ Isaac Sim 실제 상태 (10 DOF)
         │       ↳ [0~5]: joints, [7]: gripper
         │
         ├─ 호출: /compute_ik (moveit_msgs/srv/GetPositionIK) ✅ NEW
         │       ↳ XYZ+RPY → Joint 변환
         │       ↳ Cartesian Control용
         │
         ├─ 호출: /compute_fk (moveit_msgs/srv/GetPositionFK) ✅ NEW
         │       ↳ Joint → XYZ+RPY 변환
         │       ↳ Get Current Pose용
         │
         ├─ 발행: /dsr_moveit_controller/joint_trajectory
         │       ↳ trajectory_msgs/JointTrajectory
         │       ↳ MoveIt2에 궤적 명령
         │
         └─ 발행: /gripper_input (std_msgs/Float32) ✅
                 ↳ 그리퍼 제어 (라디안)

┌──────────────────────┐
│  MoveIt2             │  (모션 플래닝 & 제어)
│  - 궤적 계획         │
│  - 충돌 회피         │
└──────────────────────┘
         │
         │ 발행: /joint_states (sensor_msgs/JointState)
         ↓       ↳ 로봇 관절 상태 (6개)

┌──────────────────────┐
│ joint_state_bridge.py│  (ROS2 브릿지 노드)
│  - 메시지 변환       │
│  - 그리퍼 통합 ✅    │
└──────────────────────┘
         │
         ├─ 구독: /joint_states (sensor_msgs/JointState)
         │       ↳ 6개 관절 위치
         │
         ├─ 구독: /gripper_input (std_msgs/Float32) ✅ NEW
         │       ↳ 그리퍼 값 (라디안)
         │
         └─ 발행: /joint_input (std_msgs/Float32MultiArray)
                 ↳ 10개 DOF ✅ 확장
                 ↳ [0~5]: joints, [6]: 0, [7]: gripper, [8~9]: 0

┌──────────────────────┐
│ e0509_jointpos_ros2.py│ (Isaac Sim 시뮬레이션)
│  - 물리 시뮬레이션   │
│  - 로봇 제어         │
│  - 그리퍼 제어 ✅    │
└──────────────────────┘
         │
         ├─ 구독: /joint_input (std_msgs/Float32MultiArray)
         │       ↳ 10개 DOF
         │       ↳ index[7]을 그리퍼로 자동 처리
         │
         └─ 발행: /joint_pos (std_msgs/Float32MultiArray)
                 ↳ 현재 로봇 + 그리퍼 상태
                 ↳ 10개 DOF
```

---

## 상세 토픽 정보

### 1. `/joint_states`
- **타입**: `sensor_msgs/JointState`
- **발행자**: MoveIt2
- **구독자**: 
  - `gui_moveit.py` (MoveIt2 명령용)
  - `joint_state_bridge.py` (Isaac Sim 전달)
- **데이터 구조**:
  ```python
  {
    'name': ['joint_1', 'joint_2', 'joint_3', 'joint_4', 'joint_5', 'joint_6'],
    'position': [0.0, -0.5, 0.5, 0.0, 0.5, 0.0],  # 라디안
    'velocity': [...],
    'effort': [...]
  }
  ```

### 2. `/joint_pos` ✅
- **타입**: `std_msgs/Float32MultiArray`
- **발행자**: `e0509_jointpos_ros2.py`
- **구독자**: `gui_add_cartesian_control.py` (실시간 모니터링)
- **발행 주기**: 30Hz (성능 최적화)
- **데이터 구조**:
  ```python
  {
    'data': [
      joint_1,    # index 0, 라디안
      joint_2,    # index 1, 라디안
      joint_3,    # index 2, 라디안
      joint_4,    # index 3, 라디안
      joint_5,    # index 4, 라디안
      joint_6,    # index 5, 라디안
      0.0,        # index 6 (unused)
      gripper,    # index 7, 라디안 ✅ 그리퍼
      0.0,        # index 8 (unused)
      0.0         # index 9 (unused)
    ]
  }
  ```

### 3. `/gripper_input` ✅
- **타입**: `std_msgs/Float32`
- **발행자**: `gui_add_cartesian_control.py`
- **구독자**: 
  - `joint_state_bridge.py` (Isaac Sim 전달용)
  - `e0509_jointpos_ros2.py` (직접 제어용)
- **데이터 구조**:
  ```python
  {
    'data': 0.55  # 라디안 (0~1.101 rad = 0~63.1도)
  }
  ```

### 4. `/joint_input` ✅
- **타입**: `std_msgs/Float32MultiArray`
- **발행자**: `joint_state_bridge.py`
- **구독자**: `e0509_jointpos_ros2.py`
- **데이터 구조**:
  ```python
  {
    'data': [
      joint_1,    # index 0, 라디안
      joint_2,    # index 1, 라디안
      joint_3,    # index 2, 라디안
      joint_4,    # index 3, 라디안
      joint_5,    # index 4, 라디안
      joint_6,    # index 5, 라디안
      0.0,        # index 6 (unused)
      gripper,    # index 7, 라디안 ✅ 그리퍼
      0.0,        # index 8 (unused)
      0.0         # index 9 (unused)
    ]
  }
  ```

### 5. `/dsr_moveit_controller/joint_trajectory`
- **타입**: `trajectory_msgs/JointTrajectory`
- **발행자**: `gui_add_cartesian_control.py`
- **구독자**: MoveIt2 컨트롤러
- **사용처**: Joint Control / Cartesian Control (IK 후)
- **데이터 구조**:
  ```python
  {
    'joint_names': ['joint_1', 'joint_2', 'joint_3', 'joint_4', 'joint_5', 'joint_6'],
    'points': [
      {
        'positions': [0.0, -0.5, 0.5, 0.0, 0.5, 0.0],  # 라디안
        'time_from_start': Duration(sec=2, nanosec=0)
      }
    ]
  }
  ```

---

## ROS2 Services ✅ NEW

### `/compute_ik` (Inverse Kinematics)
- **타입**: `moveit_msgs/srv/GetPositionIK`
- **제공자**: MoveIt2
- **호출자**: `gui_add_cartesian_control.py`
- **용도**: XYZ + RPY → Joint 변환
- **호출 방식**: 비동기 (callback)

### `/compute_fk` (Forward Kinematics)
- **타입**: `moveit_msgs/srv/GetPositionFK`
- **제공자**: MoveIt2
- **호출자**: `gui_add_cartesian_control.py`
- **용도**: Joint → XYZ + RPY 변환
- **호출 방식**: 동기 (2초 timeout)

---

## 데이터 흐름

### 1. Joint Control 모드 (탭 기반)

```
GUI: Joint Control 탭 선택
   ↓
실시간 모드 ON
   ↓
슬라이더로 각도 조절
   ↓
Send Joint Command 클릭
   ↓
/dsr_moveit_controller/joint_trajectory 발행 (0.5초 duration)
   ↓
MoveIt2: 궤적 계획 및 실행
   ↓
/joint_states 발행 (실시간)
   ↓
Bridge: /joint_states → /joint_input 변환
   ↓
Isaac Sim: 로봇 제어
   ↓
/joint_pos 발행
   ↓
GUI: 모니터 업데이트 (슬라이더는 고정)
```

### 2. Cartesian Control 모드 (탭 기반) ✅ NEW

```
GUI: Cartesian Control 탭 선택
   ↓
실시간 모드 ON
   ↓
Get Current Pose 클릭
   ↓
/compute_fk service 호출 (동기)
   ↓
MoveIt2: FK 계산 (link_6 위치)
   ↓
XYZ + Quaternion 반환
   ↓
GUI: Quaternion → Euler 변환 → 입력 필드에 표시
   ↓
사용자: XYZ/RPY 수정
   ↓
Send Cartesian Command 클릭
   ↓
GUI: Euler → Quaternion 변환
   ↓
/compute_ik service 호출 (비동기)
   ↓
MoveIt2: IK 계산
   ↓
Success: Joint 값 반환 / Fail: 에러 팝업
   ↓
/dsr_moveit_controller/joint_trajectory 발행 (2.0초 duration)
   ↓
(이후 Joint Control과 동일)
```

### 3. Gripper 제어 ✅

```
GUI: 그리퍼 슬라이더 조작
   ↓
ROS Node: 점진 제어 (10Hz, 0.05 rad/tick)
   ↓
/gripper_input 발행 (Float32, 라디안)
   ↓
Bridge: 수신 후 /joint_input[7]에 통합
   ↓
Isaac Sim: 그리퍼 joint 제어
   ↓
/joint_pos[7] 발행
   ↓
GUI: 실시간 모니터링
```

### 4. 실시간 모드 특징 ✅

```
Real-time Mode OFF:
  - 슬라이더가 /joint_pos 피드백으로 자동 업데이트
  - 시퀀스 모드 활성화

Real-time Mode ON:
  - 슬라이더 자동 업데이트 중지 (사용자가 직접 조작)
  - 모니터 레이블은 계속 업데이트
  - 탭으로 Joint/Cartesian 모드 선택
  - 상호 배타적 제어 (한 번에 하나만)
```

### 4. 시퀀스 제어 (확장) ✅

```
GUI: 시퀀스 편집
   ↓
[JOINT] 블록 + [GRIPPER] 블록
   ↓
Play 버튼 클릭
   ↓
순차 실행:
  1. Joint 액션 → MoveIt2
  2. 대기 (duration)
  3. Gripper 액션 → /gripper_input
  4. 대기 (duration)
  5. 다음 액션...
   ↓
완료
```

---

## 단위 변환

| 위치 | Joint (0~5) | Gripper (7) |
|------|-------------|-------------|
| **GUI 입력** | 도 (degree) | 도 (degree) |
| **GUI 내부** | 라디안 | 라디안 |
| **/joint_states** | 라디안 | - |
| **/gripper_input** | - | 라디안 ✅ |
| **/joint_input** | 라디안 | 라디안 ✅ |
| **/joint_pos** | 라디안 | 라디안 ✅ |
| **Isaac Sim** | 라디안 | 라디안 ✅ |

**변환 공식**:
- 도 → 라디안: `rad = deg * π / 180`
- 라디안 → 도: `deg = rad * 180 / π`

**그리퍼 범위**:
- 도: 0° ~ 63.1°
- 라디안: 0.0 ~ 1.101 rad

---

## 노드별 역할 요약

### gui_moveit.py (대폭 확장) ✅
- **역할**: 사용자 인터페이스
- **기능**:
  - 포즈 편집 (각도 입력)
  - 그리퍼 제어 (슬라이더)
  - 실시간 제어 모드
  - 시퀀스 저장/로드 (Joint + Gripper)
  - 라이브러리 관리
  - 실시간 상태 모니터링
- **ROS2 연결**:
  - 구독: `/joint_states`, `/joint_pos` ✅
  - 발행: `/dsr_moveit_controller/joint_trajectory`, `/gripper_input` ✅

### joint_state_bridge.py (확장) ✅
- **역할**: 메시지 타입 변환 + 그리퍼 통합
- **기능**:
  - JointState → Float32MultiArray 변환
  - 그리퍼 값 통합 (index 7)
  - 10개 DOF 배열 생성
- **ROS2 연결**:
  - 구독: `/joint_states`, `/gripper_input` ✅
  - 발행: `/joint_input` (10개)

### e0509_jointpos_ros2.py (변경 없음)
- **역할**: Isaac Sim 시뮬레이션 인터페이스
- **기능**:
  - 관절 위치 명령 적용
  - 그리퍼 제어 (자동, index 7)
  - 실시간 시뮬레이션
  - 현재 상태 피드백
- **ROS2 연결**:
  - 구독: `/joint_input` (10개)
  - 발행: `/joint_pos` (10개)

---


## 실행 순서

```bash




# 1. MoveIt2
source /opt/ros/jazzy/setup.bash 
cd ~/ros2_ws
source install/setup.bash
ros2 launch dsr_bringup2 dsr_bringup2_moveit.launch.py mode:=virtual model:=e0509 host:=127.0.0.1

# 2. Bridge
source /opt/ros/jazzy/setup.bash 
cd ~/ros2_ws/src/doosan-robot2/isaacsim_connect/e0509_with_env_pick_place
python3 joint_state_bridge.py

# 3. GUI (Cartesian Control 포함)
source /opt/ros/jazzy/setup.bash 
cd ~/ros2_ws/src/doosan-robot2/isaacsim_connect/e0509_with_env_pick_place
python3 gui.py

# 4. Isaac Sim 
source /home/woo/IsaacSim-ros_workspaces/build_ws/jazzy/jazzy_ws/install/local_setup.bash
source /home/woo/IsaacSim-ros_workspaces/build_ws/jazzy/isaac_sim_ros_ws/install/local_setup.bash

cd ~/isaac-sim
./python.sh ~/ros2_ws/src/doosan-robot2/isaacsim_connect/e0509_with_env_pick_place/isaacsim_e0509_surface_gripper_in_usd_pick_all.py


```

## 참고 문서

- `GRIPPER_INTEGRATION_SUMMARY.md` - 통합 요약
- `ROS2_TOPIC_DIAGRAM.md` - 이 파일
- `README_GRIPPER.md` - 그리퍼 사용법
- `gui_config.py` - GUI 설정

---
