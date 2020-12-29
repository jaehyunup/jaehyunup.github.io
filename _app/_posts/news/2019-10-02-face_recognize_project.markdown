---
layout:     post
title:      OpenFace를 이용한 얼굴 학습/인식 REST API 개발기(python,Tensorflow,flask-restful)
category:  posts
tags: Developement
---

<!-- Start Writing Below in Markdown -->

### 개발 계기
----
저번학기 프로젝트 진행중 스마트 홈 시스템을 프로젝트로 하여 내부적으로 얼굴인식을 통한 보안기능을 넣었었다.

하지만 스마트홈 특성상 MCU 기기(프로젝트에서는 Raspberry Pi 3 B+ Model) 를 사용하는 환경이 대부분이고 이런 환경구성에서 Image Processing과 머신러닝을 사용한다는 것은 불가능 했다
왜냐..**너무 느려!**

혹시나 하는 마음에 OpenFace에서 제공하는 얼굴인식 모듈을 어찌어찌 rasbian에 설치했다.
실행하니 한 두개의 이미지만 30분 비교하다가 메모리 부족으로 에러가 뜬다..에효

그래서 이 OpenFace를 활용한 얼굴인식 부분만 REST API로 개발해서 GTX 1060이 달린 내 개인 PC에 연산을 맡길것이다. 

카카오나 네이버에서 제공하는 RESTAPI의 그것과 같은 서비스들을 만들어보고싶다고 항상 생각했었는데 이참에 REST API도 만들어보고 마이크로서비스가 어떻게 구성되어지는가에 대한 이해도 겸할수 있을것 같아 좋은 경험일것 같았다.


<br>
<br>

혹여나 일반적인 리눅스환경에서 OpenFace모듈을 사용하고싶다면 
[OpenFace로 우리 오빠들 얼굴 인식하기](https://www.popit.kr/openface-exo-member-face-recognition/) 위 링크를 보길 바란다.
이분이 OpenFace의 디렉토리 구조까지 아주 잘 설명해뒀다.

<br>

---

### OpenFace

[Openface](https://cmusatyalab.github.io/openface/)는 얼굴 유사도측정 오픈소스이다.
링크의 openface doc을 보면 내부가 어떻게 구성되어있는지 잘 설명이 되어있다.

한번 살펴보겠다.



## OVERVIEW
> 1. Detect faces with a pre-trained models from dlib or OpenCV.
> 
> 2. Transform the face for the neural network. This repository uses dlib's real-time pose estimation with OpenCV's affine transformation to try to make the eyes and bottom lip appear in the same location on each image.<br>
> 
> 3. Use a deep neural network to represent (or embed) the face on a 128-dimensional unit hypersphere. The embedding is a generic representation for anybody's face. Unlike other face representations, this embedding has the nice property that a larger distance between two face embeddings means that the faces are likely not of the same person. This property makes clustering, similarity detection, and classification tasks easier than other face recognition techniques where the Euclidean distance between features is not meaningful.

영어 해석하기 귀찮으시죠..? 이 남자는 무료로 해드립니다

(대신 간단하게!)
1. openface는 Dlib 또는 OpenCV를 이용해서 훈련된 모델로 얼굴 영역을 감지한다.
2. 감지된 얼굴영역을 OpenCV의 [아핀변환](https://docs.opencv.org/2.4/doc/tutorials/imgproc/imgtrans/warp_affine/warp_affine.html)과 Dlib의 [실시간 얼굴포즈 추론](http://blog.dlib.net/2014/08/real-time-face-pose-estimation.html)을 이용해서 모든 사진에서의 얼굴 정규화를 한다(즉, 같은 각도를 가진 얼굴이 될수있게끔 정규화를 시킨다)
3. 딥러닝을 이용해서 얼굴을 128-Embedding Vector로 표현한다.

<br>
정규화 과정(얼굴포즈 추론과 아핀 변환) 은 다음과 비슷하게 이루어진다<br>

<img src="/img/affine_transformation.png" width="300" style="display:block;margin-left:auto; margin-right:auto;margin-top:20px;"/>출처 - Openface


<br>

그렇다 openface의 기학습된 딥러닝 모델에 얼굴 정규화과정만 거쳐서 predict한 결과는 
128-embedding Vector로 표현되어지고 그 결과는 그 사람만의 128-embedding Vector인것이다.

즉, 얼굴이 완전 다르게 생긴 철수와 영희의 얼굴사진을 Openface의 딥러닝 모델에 넣고 predict한 결과인 철수의 embedding Vector와 영희의 embedding Vector는 멀리 떨어져 표현된다는 것이다.

반대로 얼굴이 비슷하게 생긴 철수와 철구의 얼굴사진을 넣어 predict한 결과로 얻은 두개의 128-embedding vector는 유클리드 공간상의 거리가 가깝다!(얼굴이 비슷하니까)


이제 우린 OpenFace가 어떤식으로 얼굴의 유사도 측정을 하는지 알게 되었다.



-----
### OpenFace 사용하기

<img src="/img/openface_artifact.png" width="600" style="display:block;margin-left:auto; margin-right:auto;margin-top:20px;">

출처- https://blog.algorithmia.com/understanding-facial-recognition-openface

<br>
Openface는 Torch 기반으로 학습시킨 기학습된 Neural NetWorkModel을 제공중이다.
그것이 그림에서 보라색 박스로 표현된 부분이고, 결과적으로 학습된 뉴럴네트웍을 제공해주는데 사용자가 사용하기 위해서는, 위에서 말한 정규화 Processing을 거치고 기학습되어 제공되는 NeuralNetwork에 Predict한 결과로 Classification을 하여 얼굴 정확도를 검증하는것이다.

그렇다면 위에서 우리가 구현해야할 부분은 빨간색 박스 부분이다.

저부분을 구현해서, 사용자의 Embedding Vector를 저장하고, 얼굴 인식 요청이 왔을때
현재 카메라에 감지되는 사람과 저장된 Embedding Vector 리스트를 Classification  **저장된 임베딩벡터와의 유사도가 아주높다고 판단되었을때만** True를 반환해주는 REST API로 구현할 것이다.





-----
### REST API 소개
위에서 설명한 것을 토대로 REST API를 개발했는데 사용자가 이용하는 유스케이스로 간단하게 표현한 구조는 아래와 같다.

<img src="/img/rest_artifact.png" width="800" style="display:block;margin-left:auto; margin-right:auto;margin-top:20px;">

##### 작동순서
1. 사용자는 REST API에 카메라 영상을 포함하여 얼굴인식요청을 한다
2. 요청을 받은 서버는 카메라 영상에서 얼굴 영역을 인식한다(Haar Cascade 이용) 
3. 인식된 얼굴영역을 자른다
4. Dlib을 이용해 얼굴의 68-landmark을 구분하고 68-landmark중 코에 해당하는 Landmark부분을 중앙으로 오게하여 이미지의 중앙에 코가 올수있도록 얼굴 위치를 이동시킨다.
5. 이 이미지를 OpenFace에서 기학습한 Neural Network Model을 이용해 Predict값을 받는다
6. Embedding Vector를 통해 얼굴 유사도를 측정한다 


##### 구현결과

<img src="/img/parkjaehyun.png" width="150" style="display:block;margin-left:auto; margin-right:auto;margin-top:20px;">
<img src="/img/embedding.gif" width="400" style="display:block;margin-left:auto; margin-right:auto;margin-top:20px;">

RESTAPI 요청이나 그런 과정들은 생략하였다. 글이 너무 길어질까봐..

요청이 들어왔을때 RESTAPI를 가동중인 서버는 임베딩 벡터간의 거리를 측정하고, (일반적으로 0.6~7정도 아래로 내려가면 거의 같은사람이라고 보면 된다고 한다)
해당되는 사람이 발견된다면 위와같이 Json형태로 결과를 반환해준다.


##### 소스코드
내 깃허브에 webServer.py를 보면 모든 구조가 잘 정리되어있다. 
[https://github.com/jaehyunup](https://github.com/jaehyunup/raspberrypi_smarthomeproject/blob/master/face_recognizer/webServer.py)
<br><br>

-----
### 마치는 글

어떻게 보면 간단하지만 국내 자료가 별로 없어서 꽤 오래걸렸었다
정리를 한 이유도 국내에서 처음 Openface를 사용하실분들의 불편함을 알기때문에..그런 불편함을 조금이나마 덜어보고자 정리를 했다 물론 엄청 자세하진 않지만ㅠㅠ

특히나 torch로 구현된 openface의 모델을 Tensorflow기반으로 구현하는데도 고생을 좀 했다. 

또, 사실 REST API라 하기도 좀 뭐한 REST API라서 부끄럽기도 하다.
그래도 불가능한 문제를 네트웍을 통해 클라우드서비스와 비슷하게 구현하여 해결해냈다는 것에 만족하는 프로젝트였다ㅎㅎ.
