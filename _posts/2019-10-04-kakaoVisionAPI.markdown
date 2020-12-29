---
title: "카카오 Vision REST API 사용기(python,requests)"
date: 2019-10-04
categories:
  - Development
tags:
  - ComputerVision
---

### 여담
-----
평소에 비전쪽을 많이 했었는데 마침 VISION API를 사용하게 되어 적는 글이다

(Python에서 사용하는 예제가 없기에 기본적인 사용법 정도를 예제로 보여주려한다.)

요즘 화두에 오른 유명 롤 BJ인 강만식씨의 방플의 여부가 궁금해서 개발자 답게 접근해보고자 그의 캠 얼굴 각도가 어떻게 변화하고있는지 한번 관측하고 싶어 사용했다.
**정말 의미있는 결과가 나올 수 있다면 더 더욱 좋다** 내 능력으로 사람들의 궁금한 부분을 긁어줄 수 있는거니까. 하지만 캠 각도나 화질이 부족하기때문에 잘 될 진모르겠다.

<br><br><br>

### KAKAO VISION API
-----
<img src="/img/face_demo2.jpg" width="350" style="display:block;margin-left:auto; margin-right:auto;margin-top:20px;">

<br>
<br>

카카오 VISION API는 REST API 으로 제공되어지고있다.
원하는 파라미터로 해당 API에 요청을 날리면 답변을 해준다는것이다.


카카오 API를 사용하기위해선 우선 카카오 개발자 사이트에 들어가서 API키를 발급받아야한다. 


[카카오 API키 발급 공식 레퍼런스 보기](https://developers.kakao.com/docs/restapi#앱-생성)

위 카카오 개발자 사이트를 참고하여 따라가다 보면 키가 발급될 것이다 
우리는 VISION API를 이용할 것이니 **REST API 키를 잘 기억해둬야 한다**

<br>
<br>

### OpenCV-python, Requests 모듈 설치

이미지 처리를 위해 OpenCV를 설치하고 REST API 에게 요청을 보내고 받기위해 requests 모듈을 설치한다. 

```shell
 pip install opencv-python
 pip install requests
```
<br>
<br>

### 동영상을 REST API에 보내보자
-----
동영상을 프레임마다 잘라서 계속 REST API에 요청을 할건데 사실 이건 미친짓이다. 
카카오 개발자 분들이 아시면 화낼듯ㅎㅋ

OpenCV의 videoCapture 객체를 Read하면서 while문을 통해 계속 요청을 보낼건데 이렇게 되면 너무 잦은 요청이 일어남..ㅎㅎ 하지만 테스트용이니까 괜찮죠 뭐

그럼 우선 REST API와 요청과 응답을 주고받는 방법을 알아보자
<br>
<br>

### KAKAO VISION API와 요청 응답 주고받기
-----

```python
import requests
import cv2
import glob
import sys
FACE_API_URL ='https://kapi.kakao.com/v1/vision/face/detect'
apiKey="YOUR_API_KEY"

def face_detector(filename):
    headers = {'Authorization': 'KakaoAK {}'.format(apiKey)}
    try:
        files = {'file' : open(filename, 'rb')}
        resp = requests.post(FACE_API_URL, headers=headers, files=files)
        resp.raise_for_status()
        return resp.json()
    except Exception as e:
        print(str(e))
        sys.exit(0)
```
<br>

카카오 VISION API에 요청을 하려면 다음과 같은 3가지 요소가 필요하다고 명시되어있다.
1. POST 요청이어야 한다.
2. APIKEY를 요청 헤더에 담아야 한다.
3. 파라미터로는 imageFile( JPG or PNG ) 이나 imageURL을 담아야 한다.

그걸 구현해둔것이 바로 위의 함수이다.
api 키는 적절하게 자신껄로 넣을수 있도록하면 된다

이 함수에 현재 디렉토리 기준으로 file의 path기재하면 그 파일을 파라미터로 담아 
카카오 REST API에 요청하게되며 응답으로 Json Result를 받게된다

나 같은 경우에는 이 함수를 동영상 한 프레임이 지날때 호출했다
<br>
<br>

```python
if __name__=="__main__":
    cam=cv2.VideoCapture("issue1.mp4")
    while(cam.isOpened()):
        ret,frame=cam.read()
        if ret:
            cv2.imwrite('capture.png',frame) # 이미지저장
            height,width,_=frame.shape
            files= glob.glob('*.png')
            for file in files:
                json_data=face_detector(file)
                json_result=json_data["result"]
                face=json_result["faces"]
                x=face[0]['x']
                y=face[0]['y']
                w=face[0]['w']
                h=face[0]['h']
                x=int(width*x)
                y=int(height*y)
                w=int(width*w)
                h=int(height*h)
                pitch=face[0]['pitch'] #상하회전
                yaw=face[0]['yaw'] #좌우회전
                roll=face[0]['roll']#목 기준 좌우 꺾이는지
                cv2.rectangle(frame,(x,y),((x+w),(y+h)),(0,0,255),3)
                print(yaw)
            resize_frame = cv2.resize(frame, dsize=(1080, 720), interpolation=cv2.INTER_AREA)
            cv2.imshow('KAKAO VISION API EXAMPLE', resize_frame)
            if cv2.waitKey(1) & 0xFF == ord('q'):
                break
        else:
            break
```

<br>


또, 추가로 반환받은 Json의 값을 파싱하여 해당 얼굴의 좌표에 박스를 그린다.

그리고 얼굴의 회전각까지 받은상태이다 하지만 회전각 활용은 다음에 하는것으로.


<br><br>

### 구현결과
-----
![](/img/man.png)  

<img src="/img/man.png" width="700" style="display:block;margin-left:auto; margin-right:auto;margin-top:20px;">

<br> 강만식님의 얼굴을 인식하고 회전각중 YAW를 출력하는 것을 알 수 있다.

하다보니 이미지 자체의 화질이 너무 낮아서 그런가 정확도가 다소 낮은것같았고 시간도 부족해서 여기서 그만두기로 했다. PS 공부로 너무 바빠서 신경을 더 써줄 여유가 없다..

<br>
<br>


-----
## 마치는 글
<br>

강만식님을 진심으로 응원합니다

단지 연구목적의 글임을 분명히 밝힙니다.
