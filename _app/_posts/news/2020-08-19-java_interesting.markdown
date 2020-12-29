---
layout:     post
title:      자바 짧은 팁
category:  posts
tags : Java
---
 # Study - 알쓸신잡

### 1 . boolean, byte,short, int, long, float, double, char 등등의 java primitive type들은 값의 범위가 정해져있다. 그렇다면 메모리를 효율적으로 사용하기 위해 size에 맞는 primitive type을 고민하여 써야할까? ex) -2~2 정수를 저장하기위해 byte형을 사용 해야하나?

NO, JVM의 default type은 integer 이기 때문에 오히려 JVM이 더 많은 일을 할수있는 여지를 줄 수도 있다.
  


### 2. java 의 type
JVM의 Memory allocation & management 이용 primitive type은 정해진 메모리를 할당받고 reference type은 가변적으로 메모리를 할당받는다.



### 3. 다음 코드에서 잘못된 것은?
```java
int a=10;
byte b =a;
```

명시적 type casting이 필요하다. 이때, 큰 type 에서 작은 type 으로 변수를 할당한다면 값의 손실이 일어날 수 있다.  
**큰 type <- 작은type** : 자동 형변환  
**작은 type <- 큰type**: 오류, 명시적 형변환(type casting)이 필요


### 4. 자바 byte 메모리가 다음과 같다 했을때 해당되는 숫자는?  
<center>| 1 | 0 | 0 | 0 | 0 | 0 | 0 | 1 | </center>

-128 + 1 = 127  JVM이 음수를 표현하는 방식은 다음과 같다.왼쪽 첫번째 비트는 부호비트로, 1일때 -128에 나머지 7자리의 비트의 절대값을 더해줘서 음수를 연산하는 방식이다


### 5. 비트연산자

비트연산자는 *,/ 연산자에비해 처리속도가 아주 빠르기때문에 속도나 성능면에서 critical한 문제가 일어났을때 사용할 법 하다.

### 6. Eclipse IDE 의 클래스 관리
eclipse 에서는 패키지명.class명 으로 관리하기 때문에, 물리적으로 다른 디렉토리에 있더라도 상위 패키지명이 같다면 결국 binary file화 되었을때는 같은 패키지 공간에 존재하는 클래스가 된다.



### 7. 논리연산자. 하나와 두개의 차이는 뭘까?
여러개의 논리를 엮을때 쓰는 논리연산자는 한개와 두개가 있다  &(and), |(or) 와 &&(and) , ||(and) 는 비슷하게 생겼는데 왜 별도로 분리되어 있을까?  
답은 어디까지 연산하느냐의 차이이다.  

& 와 | 같이 하나만 있는 논리연산자를 썻을때는 여러개의 조건중에 부합하는것이 있더라도 연산을 멈추지 않고 끝까지 조건을 탐색한다.  

하지만 && 과 || 같이 두번붙혀 쓰는 연산자는 여러개의 조건중에서 조건에 부합하는 분기가 나왔을때 연산을 멈추고 다음 로직을 실행하게 된다.

&와 | 를 함부로 남발한다면 성능을 저하 시킬 수 있는 요인이 될 수 있을거라 생각한다.(개인 의견)


### 8. Switch 문에는 Double 형을 사용할 수 없다.
예전부터 있지않았던 근-본 없는 double 형은 switch 문에 지원 되지 않는다.
근데 왜 String은 지원하게 해줬는데(Version 7 부터 지원) Double은 왜 안해줬지? 불쌍한 친구다.

### 9. 스택이 이용되는 대표적인 예로는 로컬변수 선언과 함수(메소드)콜 이 있다.



### 10. 잘못된 코딩 습관 바로잡기
많은 사람들이 이런 스타일로 코딩을 하고있다.
```java
if(num>2){
    System.out.println("높습니다");
}
else{
    System.out.println("낮습니다");
}
```
if 분기가 결정하는 부분은 System.out.prinln() 메서드를 실행 하냐 , 마냐가 아니라 Print 메소드 자체는 실행되면서 내부의 값만 바뀌는 것이다.  

즉 메소드의 실행은 무조건 한다는 것이기 때문에 이를 더 깔끔하게 만들 수 있는 좋은 코드 작성방식은 아래와 같다
```java
String str=""
if(num>2){
    str = "높습니다"
}
else{
    str = "낮습니다"
}
System.out.println(str);
```

### 11. 힙과 스택
코드에서 선언되어진 local variable은 스택에 저장된다. 이때, 명시적으로 값이 지정된(ex.상수 리터럴) Value type 이라면 값을 직접 가지고있고, 만약 가변적인 값이라면(new Array) Reference type이라 하고 가변적인 메모리공간을 가지는
힙이라는 공간에 값을 할당하고, 그 힙 공간의 주소값을 가지고있는다.

**상수 리터럴 타입을 선언하면 처음에 힙공간에 할당되고 나중에 또 사용되면
같은 주소값을 가지기때문에 성능에 유리할 수 도 있다** 

### 12. String 변수 값의 할당에 대한 이야기
JAVA에서 String형에 만약 "aaa"라는 문자열을 할당했다면 우선 힙에 만들어진 String constant pool(문자열 사전?해시테이블? 같은 거라고 생각하면 될것같다) 에서 찾아보고, 이 값이 없다면 pool에 새로 할당한다. 그다음 선언된 변수의 value가 constant pool에서 "aaa"라는 문자열을 가지는
는 주소값을 가르키게 된다.  

내부적으로 String 클래스의 구현을 살펴보면 intern() 이라는 메소드를 이용해 pool 을 살펴보는 과정이 구현되어있다.

그래서 아래와 같은 코드에서 String 객체간의 값을 == 연산을 통해 주소값을 비교하는데도 true라는 결과를 받아볼수 있는것이다. (pool 에서 해당하는 주소값이 같기 때문에)


```java
String a = "aaa";
String b = "aaa";
if (a == b) {
    System.out.println("true");
} else {
    System.out.println("false");
}

True
```

### 13. reference type은 4byte의 공간을 할당받는다.








