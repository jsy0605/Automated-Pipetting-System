# 🤖 Automated Pipetting System

<div align="center">
  <strong>다관절 로봇 비전 제어 및 이미지 기반 분주 용량 자동 조절 시스템</strong>
</div>

<br/>

## 📖 프로젝트 개요

생명과학 및 화학 실험에서 피펫팅(Pipetting) 작업은 높은 정밀도와 반복성을 요구합니다. 본 프로젝트는 수동 피펫과 다관절 로봇을 결합하여 실험 자동화를 구현하는 통합 제어 시스템입니다.

프로젝트의 **핵심 목표는 Well Plate의 AprilTag를 감지하여 위치와 방향을 파악하고, 피펫 팁을 지정된 Well에 정확히 정렬시켜 하강시키는 것**입니다. 이를 기반으로 마커 기반 다관절 로봇 정렬 기능을 수행하며, 더 나아가 이미지 기반 분주 용량 판독(OCR/딥러닝 활용) 및 자동 용량 조절 알고리즘을 통한 End-Effector 제어 시나리오를 구현합니다.

<br/>

## 🏗️ 시스템 아키텍처

본 프로젝트는 크게 공간 이동을 담당하는 **매크로 제어 파트(`pipetting_system`)**와 분주 용량을 조절하는 **마이크로 제어 파트(`pipet_model`)**로 나뉘어 유기적으로 동작합니다. 시스템 안정성과 환경 충돌 방지를 위해 두 시스템의 실행 환경을 분리하여 운용합니다.

### 1. 로봇 위치 정렬 시스템 (`pipetting_system`)
다관절 로봇의 End-Effector 위치를 정밀하게 이동시키고 팁을 Well에 정렬합니다.
- **비전 기반 보정:** End-Effector에 장착된 카메라가 AprilTag를 인식하여 3차원 위치와 자세를 추정하고 로봇 좌표계로 변환합니다.
- **충돌 방지 시퀀스:** 단번에 목적지로 하강하지 않고 Z축을 여러 구간으로 분할하는 선형 다점 궤적(Trajectory)을 생성하여 안전한 피펫팅을 수행합니다.
- **실행 구조:** Tkinter GUI와 ROS 2 노드가 단일 프로세스에서 동작하여 운영자의 명령과 로봇의 움직임을 즉각적으로 연결합니다.

### 2. 분주 용량 판독 및 조절 시스템 (`pipet_model`)
수동 피펫의 현재 눈금을 읽고, 목표 용량에 맞춰 물리적 다이얼을 회전시킵니다.
- **OCR 판독:** 프레임 캡처 후 YOLO로 숫자 영역(ROI)을 검출하고, TensorRT 또는 PaddleOCR을 통해 현재 설정된 용량을 판독합니다.
- **Run-to-Target 피드백 제어:** 목표 용량과 판독된 현재 용량의 오차를 계산한 뒤, 시리얼 통신(UART)으로 액추에이터에 패킷을 전송해 다이얼을 자동 조절합니다.
- **실행 구조:** GUI(PyQt5)는 시스템 파이썬 환경에서 구동되며, 딥러닝 비전 연산(Worker)은 Conda 환경에서 독립적인 프로세스로 호출됩니다.

<br/>

## 📊 성능 및 사양 (Specifications)

| 항목 | 상세 사양 |
| :--- | :--- |
| **위치 보정 정확도** | ±1mm |
| **분주 용량 OCR 정확도** | 85~90% |

<br/>

## 🛠️ 기술 스택

### Hardware
- **Robot:** UR Robot (다관절 로봇)
- **Camera:** Allied Vision Alvium 1800 U-511c Camera
- **Actuators:** Linear Actuator & DC Motor

### Middleware & Vision SDK
- ROS 2 (Jazzy)
- MoveIt 2
- Vimba SDK (VmbPy)

### Languages & Environments
- **Python 3.12:** 로봇 제어, ROS 2 노드, 메인 시스템 연동 (`pipetting_system`)
- **Python 3.10:** YOLO 검출, OCR 판독 및 딥러닝 모델 Worker (`pipet_model`)

### AI & Computer Vision
- OpenCV, pupil-apriltags (마커 인식)
- YOLO (ROI 검출)
- TensorRT, PaddleOCR (이미지 기반 숫자 인식)

<br/>

## 🚀 시작하기 (Getting Started)

### 1. 로봇 제어 시스템 실행 (`pipetting_system`)
Python 3.12 및 ROS 2 환경에서 실행합니다.

```bash
# Terminal 1: UR Driver 실행
ros2 launch ur_robot_driver ur_control.launch.py ur_type:=ur5e robot_ip:=<ROBOT_IP>

# Terminal 2: MoveIt 실행
ros2 launch ur_moveit_config ur_moveit.launch.py ur_type:=ur5e

# Terminal 3: 메인 GUI 실행
cd pipetting_system
python3 main.py
```

### 2. 용량 제어 및 비전 시스템 실행 (`pipet_model`)
GUI는 시스템 파이썬에서, 비전 연산은 Python 3.10 기반의 Conda 환경(pipet_env)에서 실행됩니다.

```bash
# 시스템 환경에서 메인 GUI 실행
cd pipet_model/ocr_motor
python3 -m gui.main
```

**Note**: YOLO 및 OCR Worker는 GUI 내에서 conda run -n pipet_env python -m worker.worker 형태로 자동 호출됩니다.

## 📌 운영 간 유의사항
1. **툴 장착 각도 (Mount Angle)**: 로봇이 예상과 다르게 움직일 경우 가장 먼저 config.py의 MOUNT_ANGLE_DEG (기본 -60°) 설정과 축 변환 로직을 점검해야 합니다.

2. **프로세스 분리 원칙**: PyQt5 환경(시스템)과 TensorRT/CUDA 환경(Conda)을 강제로 병합할 경우 라이브러리 충돌이 발생할 수 있으므로 분리된 아키텍처를 유지해야 합니다.

3. **영점 보정**: 카메라의 픽셀 오차를 기반으로 로봇 좌표를 근사하므로, 태그 사이즈나 렌즈 초점 거리 등 하드웨어 변경 시 관련 파라미터를 반드시 재설정해야 합니다.
