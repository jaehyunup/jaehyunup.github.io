---
title: "커뮤니케이션 티칭 웹 서비스"
date: 2019-11-13
categories:
  - Development
tags:
  - ComputerVision
  - Web
---

### 들어가는 글
-----
감정인식.. 얼굴인식과 비슷한 맥락이다
CNN을 이용하고 [fer2013](https://www.kaggle.com/c/challenges-in-representation-learning-facial-expression-recognition-challenge/data) Dataset을 이용해서 학습을 시켰다. 

모델의 구성은 다음과 같다

![](/img/emotion_model.png)  


학습에 대한 과정은 [https://github.com/gitshanks/fer2013](https://github.com/gitshanks/fer2013) 이분의 github를 참고를 많이했고 내가 직접 학습시킬때 보다 훨씬 더 좋은 결과를 낼 수 있었다.


---
> **내가 개발하고있는 감정인식 모듈의 파이프라인은 다음과 같다.**
> 1. Video Frame read (OpenCV)
> 2. 얼굴 영역 인식(Dlib FaceDetector)
> 3. 얼굴 영역을 Crop
> 4. Crop된 얼굴 영역을 미리 학습해둔 학습모델에 Predict.(Keras)
> 5. 반환값을 이용해 감정의 변화를 확인하고 피드백 해줌

현재 내가 개발하고있는 모듈의 상태는 5단계인 **반환값을 이용해 감정의 변화를 확인하고 피드백** 을 제공하는 수준을 바라보고있다. 


![](/img/emotion_test.png)


요청된 비디오에따라 감정 변화와 프레임별 감정을 기록하고 이 정보를 통해서 화자의 감정표현이 올바른지, 아닌지 판별하고 피드백을 줄수있게끔 준비해둔 상태이다.

이제 이 감정들을 얼마나 잘 활용하는지가 문제인데.. 조금 더 고민해봐야할 문제인것같다

이제 감정인식은 개발되었고 다음으로 할것은 **음성어조분석, 자연어 분석을 통한 키워드 찾기** 정도가 남았다고 할 수있겠다. 

이제 자연어분석을 공부해야겠다.








