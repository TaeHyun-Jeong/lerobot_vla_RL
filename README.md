# 🤖 VLA & RL 기반 작업 환경 어시스턴트 로봇

> 🥇 **2026 국민대학교 전자공학 창의설계 경진대회 금상**

VLA(Vision-Language-Action)와 강화학습(RL)을 결합하여 **주변 작업 환경을 인식하고, 상황에 맞는 정리 작업을 스스로 결정하여 수행하는 로봇**을 구현했습니다.

기존 VLA가 사용자가 지정한 작업을 수행하는 데 집중했다면, 본 프로젝트에서는 **RL을 상위 의사결정 계층으로 추가하여 무엇을 정리할지 결정**하도록 구성했습니다.

- **YOLO** → 작업 환경 인식
- **RL Model** → 물체와 수납 위치 결정
- **VLA** → 결정된 작업을 실제 로봇 동작으로 수행
- **LeRobot** → 실제 로봇 제어 및 데이터 수집

---

## 🎥 Final Demo

![Final Demo](assets/final_demo.gif)

> 최종 시연에서는 **정리 모드(Organizing Mode)**를 중심으로 전체 Pipeline을 구현했습니다.

---

# 📌 Project Overview

일반적인 VLA 기반 로봇은 사용자가 지정한 작업을 어떻게 수행할 것인가에 집중합니다.

```text
"Battery를 gray bin에 넣어줘."
        ↓
      VLA
        ↓
   Robot Action
```

하지만 실제 작업 환경에서는 사용자가 **어떤 물체를 어느 수납공간에 넣을지 일일이 명령해야 하는 불편함**이 발생합니다.

이를 해결하기 위해 **RL을 상위 의사결정 계층으로 추가하여 정리 작업을 스스로 결정**하도록 구성했습니다.

```text
Camera
  ↓
YOLO Detection
  ↓
RL Observation
  ↓
RL Model
  ↓
Object → Target Bin
  ↓
Natural Language Command
  ↓
Object-specific VLA Policy
  ↓
LeRobot Robot Arm
```

즉,

> **RL은 무엇을 할지 결정하고, VLA는 결정된 작업을 실제 로봇 동작으로 수행합니다.**

---

# 🏗️ System Architecture

```text
Camera
  ↓
YOLO
  ↓
RL Observation
  ↓
┌──────────────────────┐
│      RL Model        │
│  + Action Masking    │
└──────────┬───────────┘
           ↓
    Object + Target Bin
           ↓
 Natural Language Command
           ↓
┌──────────────────────┐
│ Object-specific VLA  │
└──────────┬───────────┘
           ↓
      LeRobot Arm
```

---

# 🧠 Hierarchical RL + VLA

본 프로젝트의 핵심은 **RL과 VLA의 역할을 분리한 계층적 구조**입니다.

### RL — High-Level Decision

RL은 현재 환경을 바탕으로

- 어떤 물체를 선택할지
- 어느 수납공간으로 이동시킬지

를 결정합니다.

Action은 다음과 같이 구성했습니다.

```text
Action = Object × Target Bin
```

4개의 물체와 4개의 수납공간을 기준으로 총 **16개의 Action**을 구성했으며, Action은 **One-Hot Encoding** 형태로 표현했습니다.

### RL → VLA

RL 모델이 정리할 물체와 수납공간을 결정한 뒤, **해당 결정을 자연어 작업 명령으로 변환하여 출력**합니다. 생성된 명령은 물체별 VLA Policy에 전달되어 실제 로봇 동작으로 수행됩니다.

```text
RL Decision
      ↓
Object + Target Bin
      ↓
Natural Language Command
      ↓
Object-specific VLA Policy
      ↓
Robot Execution
```

---

# 🎯 Custom RL Environment

실제 로봇에서 바로 강화학습을 수행하는 대신, **Gymnasium 기반의 Custom RL Environment를 직접 구성**하여 RL 모델을 학습했습니다.

환경에는 다음 요소를 포함했습니다.

- 4개의 물체
- 4개의 수납공간
- 물체 위치 및 존재 여부
- 수납공간의 현재 무게
- 물체별 무게 및 수납공간 최대 용량
- 로봇 팔 위치

---

# 📐 State

RL은 이미지 자체를 입력으로 사용하는 대신, 현재 작업 환경을 **수치 벡터 형태의 Observation**으로 변환하여 사용합니다.

각 물체는 다음과 같이 표현합니다.

```text
(x, y, existence)
```

4개의 물체 정보와 4개의 수납공간 무게를 사용하여 최종 Observation을 **16차원 벡터**로 구성했습니다.

```text
Object Positions & Existence : 12
Bin Weights                  :  4
                              ----
                              16
```

---

# 🎮 Action

Action은 **어떤 물체를 어떤 수납공간에 넣을 것인가**를 나타냅니다.

```text
4 Objects × 4 Bins = 16 Actions
```

예:

```text
Action 5
→ Object 1
→ Bin 1
```

Action은 **One-Hot Encoding** 형태로 표현했습니다.

---

# 🚫 Action Masking

모든 Action이 실제 환경에서 가능한 것은 아닙니다.

예를 들어,

- 이미 정리된 물체
- 존재하지 않는 물체
- 물체 크기와 수납공간 조건상 불가능한 선택

등은 선택할 필요가 없습니다.

따라서 RL 모델이 Action을 선택하기 전에 **현재 상태에서 불가능한 Action을 Masking**하여 유효한 Action만 선택하도록 구성했습니다.

---

# 🏆 Reward Design

로봇이 단순히 물체를 아무 수납공간에 넣는 것이 아니라, **짧은 이동 거리와 적절한 수납공간 선택을 고려하도록 Reward를 설계**했습니다.

주요 Reward 요소는 다음과 같습니다.

- **수납공간까지의 거리** → 이동 거리 최소화
- **Robot Arm과 물체 사이 거리** → 불필요한 이동 감소
- **수납공간의 남은 용량** → 적절한 수납공간 선택
- **무거운 물체의 우선 처리** → 정리 순서 고려
- **잘못된 수납 Penalty** → 부적절한 수납 방지

---

# 🧩 Dueling Double DQN

RL Agent는 **Dueling Network와 Double DQN을 결합**하여 구현했습니다.

### Dueling Network

State의 전체적인 가치와 각 Action의 상대적인 중요도를 분리하여 학습합니다.

```text
State
  ↓
Feature Extraction
  ├── Value Stream
  └── Advantage Stream
          ↓
       Q Values
```

### Double DQN

- **Online Network** → 다음 Action 선택
- **Target Network** → 선택된 Action 평가

방식을 사용하여 학습 안정성을 높였습니다.

---

# 👁️ YOLO 기반 환경 인식

실제 로봇 환경에서는 카메라 영상을 YOLO로 분석합니다.

```text
Camera Image
     ↓
    YOLO
     ↓
Object Detection
     ↓
Object Position
     ↓
RL Observation
```

Detection 결과는 RL에서 사용할 수 있는 Observation 형태로 변환됩니다.

---

# 🔄 실제 동작 과정

정리 모드에서는 다음 과정을 반복합니다.

```text
Camera
  ↓
YOLO Object Detection
  ↓
RL Observation
  ↓
Action Masking
  ↓
RL Decision
  ↓
Object → Bin
  ↓
Natural Language Command
  ↓
Object-specific VLA Policy
  ↓
Robot Pick & Place
  ↓
Environment Update
  ↓
반복
```

정리할 물체가 더 이상 없으면 정리 모드를 종료합니다.

---

# 🛠️ Problem Solving

## 1. Reward 간 균형 문제

거리, 수납공간 용량, 물체 무게 등을 동시에 고려하면서 특정 Reward가 지나치게 큰 영향을 주는 문제가 있었습니다.

예를 들어 거리 Reward의 영향이 지나치게 커지면 **정리의 적절성보다 이동 거리만 줄이는 방향**으로 학습될 수 있습니다.

### 해결

각 Reward의 가중치를 조절하며 특정 요소에 치우치지 않도록 **Reward 간 균형을 반복적으로 튜닝**했습니다.

---

## 2. YOLO 입력 이미지의 색상 왜곡

기존 VLA Pipeline에서 사용하는 카메라 이미지를 그대로 YOLO 입력으로 사용했을 때 **이미지 색상이 왜곡되는 문제**가 발생했습니다.

### 해결

VLA Pipeline과 RL/YOLO의 이미지 처리를 분리하고, **OpenCV를 이용한 YOLO 전용 이미지 전처리 과정**을 구성했습니다.

---

## 3. 하나의 VLA 모델로 모든 작업을 처리하기 어려운 문제

여러 종류의 정리 작업을 하나의 VLA 모델에서 처리하려 했을 때 원하는 수준의 동작 학습이 이루어지지 않았습니다.

### 해결

**물체 종류에 따라 VLA Policy를 분리**하고, 각 Policy에서 해당 물체에 대한 4가지 작업을 수행할 수 있도록 구성했습니다.

---

## 4. VLA Policy Loading에 따른 실행 지연

작업마다 VLA Policy를 새롭게 Load하면 실제 실행 과정에서 불필요한 시간이 발생했습니다.

### 해결

실제 실행 전에 필요한 VLA Policy들을 **미리 Load하여 메모리에 유지**하도록 구성했습니다.

---

## 5. 실제 로봇에서 RL을 직접 학습하기 어려운 문제

실제 로봇에서 강화학습을 수행하면 반복적인 로봇 동작으로 인해 학습 시간과 하드웨어 사용 부담이 커집니다.

### 해결

**Gymnasium 기반 Custom RL Environment에서 RL을 학습**한 뒤, 실제 환경에서는 YOLO를 통해 얻은 정보를 동일한 Observation 구조로 변환하여 학습된 Policy를 적용했습니다.

---

# 👨‍💻 My Contribution

본 프로젝트에서 **RL과 YOLO 관련 구현을 주로 담당했습니다.**

### Reinforcement Learning

- Gymnasium 기반 Custom RL Environment 구현
- State / Action 설계
- Reward 설계 및 튜닝
- Action Masking
- Dueling Network / Double DQN 구현
- RL Training / Deployment

### Computer Vision

- YOLO 기반 Object Detection
- Detection 결과의 RL Observation 변환
- YOLO 입력 이미지 전처리
- 실제 환경에서 RL과 YOLO 연동

### VLA Integration

- RL Decision → VLA Task Command 변환
- Object별 VLA Policy 선택
- Policy Preloading
- RL → VLA → LeRobot 실행 Pipeline 연동

> VLA Dataset 구축 및 전체 시스템 구성은 팀원들과 함께 진행했으며, RL 및 YOLO 구현을 담당했습니다.

---

# 🌟 Result

본 프로젝트는

> 🥇 **2026 국민대학교 전자공학 창의설계 경진대회 금상**

을 수상했습니다.

최종 시연에서는 **정리 모드(Organizing Mode)**를 중심으로 YOLO가 작업 환경을 인식하고, RL이 정리할 물체와 수납공간을 결정한 뒤, VLA가 실제 로봇 동작을 수행하는 전체 Pipeline을 구현했습니다.

---

# 📁 Project Structure

```text
lerobot_vla_RL/
│
├── web/
│   └── Web Interface
│
├── src/
│   └── lerobot/
│       ├── scripts/
│       │   ├── gym_env.py
│       │   ├── RL_train.py
│       │   ├── RL_deploy.py
│       │   ├── yolo_detect.py
│       │   ├── lerobot_record_web.py
│       │   ├── lerobot_record_daemon.py
│       │   └── tts.py
│       │
│       └── ...
└── ...
```

---

# 🚀 Execution

### RL Training

```bash
python RL_train.py
```

### RL Deployment

```bash
python RL_deploy.py
```

### LeRobot 실행

프로젝트 환경에 맞게 LeRobot 및 Web Interface를 실행한 뒤, 정리 모드를 선택하여 RL 기반 자동 정리를 수행합니다.

---

# 💡 What I Learned

### 1. Reward 설계의 중요성

단순히 Reward를 많이 받도록 만드는 것이 아니라, **어떤 행동을 유도할 것인지 정의하고 각 Reward 요소의 균형을 조절하는 과정**이 중요하다는 것을 경험했습니다.

### 2. RL과 VLA의 역할 분리

RL을 로봇의 모든 동작을 직접 제어하는 용도로 사용하는 대신,

> **YOLO → 환경 인식**  
> **RL → 의사결정**  
> **VLA → 행동 수행**

으로 역할을 분리하면서 각 모델의 장점을 활용할 수 있었습니다.

### 3. 실제 시스템에서는 Runtime 구조도 중요

여러 VLA Policy를 사용하는 과정에서 발생한 Loading 지연을 **Policy Preloading**으로 해결하면서, 실제 로봇 시스템에서는 모델 성능뿐 아니라 **Inference Pipeline과 Runtime 구조**도 중요하다는 것을 경험했습니다.
