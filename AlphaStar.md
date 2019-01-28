## AlphaStar
### 개요

![image](https://storage.googleapis.com/deepmind-live-cms/images/screenshot.width-1500.png)

1. Starcraft 2 강화학습 모델
2. 유명 해외 프로게이머 (MaNa, TLO) 을 상대로 5:0 전승 기록 (18.12.19)
3. 관련 사이트 : [딥마인드 포스트](https://deepmind.com/blog/alphastar-mastering-real-time-strategy-game-starcraft-ii/) , [유투브 영상](https://www.youtube.com/watch?v=UuhECwm31dM)

### 구성 

![animation](https://storage.googleapis.com/deepmind-live-cms/documents/sc2-agent-vis%2520%25281%2529.gif)

1. State
2. Model
3. Action

### 1. State
- __게임의 특징__
  1. Game Theory : 승리를 위한 다양한 전략이 있으며, 상대에 대응하여 전략을 수립하여야 한다. (바둑, 체스와 유사)
  2. 정보의 불완전성 : 맵 전체가 공개 되지 않고 자신의 기지와 유닛이 가본 적 있는 부분만이 관찰 가능하다.
  3. 긴 플레이 시간 : Action의 결과가 바로 주어지지 않으며, 전체적으로 1시간 가까운 긴 시간 동안의 상태 변화를 추적해야 한다.
  4. 실시간 전략 게임 : 바둑과 달리 턴제가 아니고, 상대와 자신의 플레이가 동시에 진행된다.
  5. 광범위한 action space : 조종해야 하는 기지 및 유닛이 다양하고, 각각이 수행할 수 있는 action 종류가 많다. (Multi-agent)

- __Input Process__

  - 현재 게임 화면이 모델의 입력값으로 주어지게 된다.

  - 게임화면이 3D그래픽이기 때문에, 그대로 주어지지 않고, 각 유닛 및 기지의 Feature가 반영된 화면으로 전환된다.  [(feature_layer_video)](https://www.youtube.com/watch?v=5iZlrBqDYPM)

  - 관찰 가능한 모든 지역이 한 번에 주어지지만 주의가 필요한 부분을 선택하여 판단에 반영하는 모습을 보인다.

  - 사람 플레이어와 동일하게 현재 화면 상에 보이는 정보만 주어지는 네트워크 역시 훈련되었고, 시간의 차이가 있을 뿐 비슷한 결과를 보여주었다. (유닛을 조종하여 화면을 전환 가능하다.)

    ![camera](https://storage.googleapis.com/deepmind-live-cms/images/SCII-BlogPost-Fig10_02.width-1500.png)

### Model
1. __Overview__

   - 현재 게임 상황을 입력값으로 받고, 다음에 취할 action을 출력하는 모델이다.

   - Supervised와 Reinforce를 결합한 구조를 취한다.

   - 사람 플레이어의 대전 정보를 이용하여 Supervised Model을 구축한다.

   - Laddering을 활용한 Self-play를 반복하여 Model을 강화 학습한다.

2. __StarCraft Ladder__

   ![ladder](https://storage.googleapis.com/deepmind-live-cms/images/SCII-BlogPost-Fig03.width-1500.png)

   - Population-based, Multi-agent Learning

   - Self-play를 기반으로 한 강화학습 방법

   - 여러 Agent가 존재하며, 새로운 Agent는 기존의 Agent를 이기는 것을 목표로 한다.
   - 새로운 Agent는 이전의 Agent 하나와 대전할 수도 있고, 이전 Agent들을 합친 복합 모델과 대전하기도 한다.
   - 처음에는 단순한 방법이 선호되지만 점차 복잡한 전략을 활용하게 된다.

3. __Model Application__ 
  1. Transformer units + LSTM core : self-attention 기법을 활용한 RNN 모델

  2. Auto-regressive Policy head : action의 시계열을 고려하여 현재 policy의 variablity를 줄인다. (현재 state만이 아니라 앞선 action도 조건으로 고려)

  3. Pointer Network : seq2seq기반. decoder에서 encoder의 attention vector를 softmax probability로 직접 활용하는 모델 (encoder 정보를 활용하고, decoder의 vocab size 제한 문제를 해결한다.) [arxiv](https://arxiv.org/abs/1506.03134)

     ![pointer](https://cdn-images-1.medium.com/max/800/1*ztyKI9gryzcu-26PdHGRWg.png)

  4. Centralized Value baseline : actor-critic 기반. 다수의 actor가 하나의 critic과 parameter sharing을 통하여 학습한다. (A3C의 global network와 비슷)

4. __Supervised Model__

   ![supervised](https://storage.googleapis.com/deepmind-live-cms/images/SCII-BlogPost-Fig04.width-1500.png)

   - 프로게이머의 플레이 자료를 통하여 학습한 모델
   - "엘리트" 수준의 Bot 플레이어를 상대로 95% 확률로 승리 가능 (Gold 레벨)
   - Ladder 강화학습의 baseline으로 이용된다.

5. __Reinforcement Model__
  1. Off-policy : Sample에 활용되는 Behavior policy와 Optimization에 적용되는 Target policy가 분리되어, 효율적 / 안정적 학습이 가능.
  2. Actor-Critic : Policy를 결정하는 actor network와 Q-value를 계산하는 critic network가 구분, 3-4와 연동.
  3. Experience Replay : Sampling을 통해 얻은 instance를 buffer에 저장하여, 학습에 여러 번 활용하여 선택적 / 안정적 학습이 가능하다 (off-policy와 연동)
  4. Self-imitation Learning : 결과가 좋았던 Experience를 학습 대상으로 삼는다. (Experience Buffer에서 추출)
  5. Policy Distillation : Teacher-Student Transfer. Teacher model의 output을 target으로 삼은 student model을 학습. [arxiv](https://arxiv.org/pdf/1511.06295.pdf)

  ![training](https://storage.googleapis.com/deepmind-live-cms/documents/sc2-progression%2520%25281%2529.gif)

### Action
1. __Action ouput__

   - move_screen, selec_rect, select_add 등 키 입력이 아닌 함수로 대체
   - 300개 함수와 13개의 argument로 구성
   - 불가능한 Action이 선택되는 등 error 방지를 위해 filter 적용

2. __Action Per Minute (APM)__

   ![apm](https://storage.googleapis.com/deepmind-live-cms/images/SCII-BlogPost-Fig09.width-1500.png)

   - 모델의 평균 APM은 280으로 사람 평균 (30~300) 범위 이내.
   - 프로 선수 (MaNa - 390, TLO - 678)에 비해 낮은 편으로 실제 유저 플레이 기반 학습의 영향으로 추정.