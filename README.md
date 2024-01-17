# Yolov7_with_JetsonNano
#### 팀명 : 우리가 남희가

#### 기간 : 2023. 10. 25 ~ 2023. 12. 04

#### 팀장 : 이용주

#### 팀원 : 공수찬, 강동훈, 김남희, 엄서영, 이승헌

# 아이디어 배경
- 운전자가 운전석에서 느끼는 사각지대는 많고 실제 안전에 많은 방해가 됨
- 자동차의 장착되어있는 센서로도 감지되지 않는 사각지대가 있음
- 그래서 단안카메라를 통해 이동체와 사람을 감지하여 차선 변경시 위험을 탐지할 수 있는 시스템을 고안
<img src="https://github.com/SeungHeon3649/Yolov7_with_JetsonNano/assets/94602281/299d63f2-7fb6-4ba1-8dac-e22ecb282e99" width="600" height="400"/>

<img src="https://github.com/SeungHeon3649/Yolov7_with_JetsonNano/assets/94602281/54f4623f-676a-4ac4-8c7c-724001490ce1" width="600" height="400"/>

# 구현 내용
- OpenCV를 통해 차선을 감지하고 주행 차로와 양 옆 차선을 구분하여 차선 변경시에 해당 차로의 위험성을 판단 할 수 있는 범위를 설정
- Yolov7을 통해 객체를 탐지, 모델 구조의 변형을 통해 객체 탐지 뿐 아니라 거리를 추정할 수 있도록 함
- 차선 변경시 진출 차로에 차량이 있는지를 판단, 옆에 차량이 없고 뒤에서 오는 차량에 대한 안전성이 확인되면 운전자에게 다시 확인후 차선을 변경하라고 청각적으로 알림
- 차선 변경시 신출 차로에 바로 옆에 차량이 있거나 뒤에서 오는 차량과 추돌 위험이 있을 경우 운전자에게 청각적으로 위험 신호를 전달

# 데이터 수집
- 이동체(차량, 오토바이, 버스 등)과 사람 데이터를 AIhub에서 수집
- 단안카메라를 통해 거리도 추정하게 할 것이므로 객체 바운딩 박스 데이터와 라이더를 통한 거리 데이터가 있는 KITTI데이터 수집
- 측면에서 바라보는 데이터가 부족하므로 직접 촬영하여 수집하고, 블랙박스 데이터도 직접 수집하여 테스트 데이터로서 활용
- 직접 수집한 데이터는 Labelimg를 사용해 직접 라벨링

# 데이터 전처리
- Yolo모델에서 학습이 가능하도록 라벨을 Yolo모델에 맞게 변형(class, x_center, y_center, width, height)
- KITTI데이터에서 거리가 추가된 Yolo모델 라벨 형태로 변형(class, x_center, y_center, width, height, distance)

# 거리추정 모델 학습
- 모델 구조를 변경하여 거리추정이 가능하도록 함
- 앙상블 모델을 사용

# 거리추정 모델 테스트
#### 1. 기존의 YOLO데이터와 거리데이터를 통해 학습 시켜서 바운딩 박스, 클래스 정보, 신뢰도와 거리가 반환되도록함
- 기존 모델에서 출력값을 하나 추가하는 방식
  
  ![image](https://github.com/SeungHeon3649/Yolov7_with_JetsonNano/assets/94602281/f42d6725-e9ac-4dc2-b54e-37319d554d06)

#### 2. 기존의 YOLO데이터와 거리데이터를 통해 학습 시켜서 바운딩 박스, 클래스 정보, 거리가 반환되도록 함
- 기존 모델에서 헤드 부분을 분할하여 하나의 헤드는 기존 헤드로서 바운딩 박스와 클래스 정보, 신뢰도를 반환 하도록함
- 다른 헤드 부분에서는 리니어한 헤드 형태로 구성하여 거리값을 도출할 수 있도록 설계함

  ![image](https://github.com/SeungHeon3649/Yolov7_with_JetsonNano/assets/94602281/f9db6027-7e14-43d0-ac9e-27925469b25a)

#### 3. 앙상블 모델
- YOLO + Linear
  
  <img src="https://github.com/SeungHeon3649/Yolov7_with_JetsonNano/assets/94602281/bc970545-5dc4-48b7-805e-8cc9c1e5bb35" width="480" height="300"/>

#### 최종 모델 : 앙상블 모델
- 위의 1, 2 번 접근 방식에서 거리에 대한 로스가 제대로 줄지 않아서 앙상블 모델로 접근

# OpneCV를 활용한 차선 인식
- 마스크를 통해 차선 검출 범위를 설정
- 설정된 범위에서 노이즈를 제거하고 엣지를 검출
- 엣지를 검출할 때 수평선에 가까운 선분은 무시
- 검출된 선분의 최하단 부터 차선의 소실점까지 연결하여 차선을 탐지
- 소실점을 활용한 차선 탐지를 활용하여 주행중인 차선과 양 옆 차선 인식 가능
- 모델을 통해 검출된 이동체와 이동체의 상대 속도, 거리에 따라 위험성을 판단
- 차선 변경시 안전하면 파란색, 위험하면 빨간색으로 시각화, 차선 변경 신호가 입력되지 않으면 초록색으로 표시하고 객체 탐지와 상대속도 추정을 실시간으로 계속 진행
- 실시간 탐지를 진행하다 신호가 입력되는 즉시 청각적으로 알람(시각화에서는 입력 즉시 파란색 혹은 빨간색으로 시각화함)

<img src="https://github.com/SeungHeon3649/Yolov7_with_JetsonNano/assets/94602281/0bdef3bf-de56-4928-a313-8cfefb3176f7" width="600" height="300"/>


<img src="https://github.com/SeungHeon3649/Yolov7_with_JetsonNano/assets/94602281/0e25e9cd-b7b6-4d83-9f0c-d571da852e7c" width="600" height="300"/>

# 하드웨어 임베딩
- 리눅스를 설치하고 리눅스 명령어를 통해서 개발환경을 구축
- VScode와 ssh를 이용해여 원격 개발 세팅을 함(원격 개발 세팅을 통해 파일을 옮기고 수정하는 작업을 수월하게 함)
- Cuda, Cudnn, Pytotch-gpu, OpenCV 등을 설치하며 개발환경 구축 마무리(버전을 잘 확인하면서 해야함)
  
![image](https://github.com/SeungHeon3649/Yolov7_with_JetsonNano/assets/94602281/69eb6282-a3b6-41c0-9547-1fba19d117e7)

# 양자화
- JetsonNano에 임베딩 하였을 때, FPS가 제대로 나오지 않음
- GPU세팅과 하드웨어 가속을 진행 하였을 때가 Yolo객체 검출만 3프레임
- 양자화를 진행한 후에 9프레임
- TensorRT를 활용해서 양자화 진행

  ![image](https://github.com/SeungHeon3649/Yolov7_with_JetsonNano/assets/94602281/b599679d-d149-489d-8828-ed3c2712378f)

  <img src="https://github.com/SeungHeon3649/Yolov7_with_JetsonNano/assets/94602281/b08dc420-995b-4598-8306-1e719083bd36" width="480" height="320"/>

# 결과

<img src="https://github.com/SeungHeon3649/Yolov7_with_JetsonNano/assets/94602281/e1395968-107c-4dee-8e52-5e0ce601822b" width="480" height="320"/>

- 이 사진과 같이 차선 변경을 위해 방향지시등을 켰을 때, 옆에 차량이 없으면서 후방에 차량의 추돌 위험성도 없으면 청각적인 신호를 통해 운전자에게 알려줌
- 해당 사진은 비쥬얼라이징을 통해 시각적으로 설명하기 위한 것으로 실제 JetsonNano에 탑제 될 때는 비쥬얼라이징하는 코드는 전부 성능을 위해 제거함

# 특이점
- JetsonNano라는 제한된 하드웨어에서 성능을 내기 위한 작업이 많이 힘들었음
- 컴퓨터에서는 잘 작동하다가 임베딩시 동작하지 않는 문제가 많이 발생, 생각보다 고려해야할 점이 많았음, 대표적으로 버전문제와 하드웨어 문제
- 프로젝트 기간 때문에 진행되었던 모델 테스트 중 가장 경향성이 잘 나왔던 앙상블 모델로 진행하였는데 기간이 조금 더 있었다면 다른 방법으로도 마무리 해보고 싶음
- 차선인식은 전적으로 OpenCV를 통해서 진행하였는데, JetsonNano라는 하드웨어 제약이 없다는 가정하에 모든 검출과 판단을 딥러닝 모델로서 작업해보고 싶음
