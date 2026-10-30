---
id: "01HG4ET100AYM7JVXNQB4X62XV"
slug: "raspberry-pi-iot-vending-machine"
title: "라즈베리파이 IoT 자판기 만들기: Firebase 재고 관리와 도난 감지"
category: "01M651PC00H3J4W7JXTZ32PR45"
tags: ["raspberry_pi","firebase","linux","python"]
summary: "데이터베이스를 이용한 IOT 자판기 설계"
thumbnail: "/img/01HG4ET100AYM7JVXNQB4X62XV/eb0f3fc7c854.webp"
note: false
created: "2023-11-26T09:00:00+09:00"
updated: "2026-09-15T02:10:29+09:00"
---
![IOT를 접목한 본격적인 자판기 using RaspberryPi|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/eb0f3fc7c854.webp)

## 1. 문제 상황 및 핵심 기능
자판기는 일상생활에서 쉽게 볼 수 있는 편의 장치 중 하나입니다. 대부분 자판기를 보면 재고가 남아 있지만, 사람들이 많이 다니지 않는 곳의 자판기를 보면 재고가 없을 때가 많습니다. 이는 수요가 없는 지역에서 자판기의 재고 관리에 쓰는 노력이 수익대비 크기 때문입니다. 그래서 <u>사람이 일일이 확인하러 다니지 않고도 온라인 database 상으로 재고를 관리할 수 있는 IOT를 접목한 자판기</u>를 고안하게 되었습니다.

또한, 외진 곳은 CCTV나 경비가 부족하다. 이러한 이유로 자판기의 물건이 도난당하는 일이 있는 데, 사이렌을 울리며 온라인 database 상으로 관리자에게 상황을 알리는 기능을 추가하여 이에 대비할 수 있습니다.

1) 자판기를 IOT로 구현함으로써, 재고가 다 떨어진 상황을 외부에서 확인할 수 있습니다. 즉, 계속해서 재고를 확인하는 인력을 줄일 수 있습니다.
2) 현재까지의 자판기 수입금을 확인하여 이후 자판기 투자(제품 선택, 자판기 위치 선택 등)에 대한 계획을 세울 수 있습니다.
3) 자판기의 물건을 훔치는 일을 막아, CCTV나 경비가 부족한 지역에서도 사용할 수 있다는 장점이 있습니다.
4) 자판기에 기본적으로 탑재된 관리자 호출 버튼을 이용하여 자판기에 문제가 발생하였을 때 대체가 가능합니다.

## 2. HW 구성

### 1) 사용 부품
라즈베리파이, 180도 서보모터, 360도 서보모터, 버저, 텍트 스위치 3개, 근접센서 3개, LED 2개, firebase 데이터 스토리지

### 2) 윗면 구성
자판기는 크게 메인 system을 구성하는 왼쪽 부분과 출력을 담당하는 오른쪽 부분으로 나눌 수 있습니다. 출력부는 뚜껑이 열리게 설계하여 물건을 보충할 수 있도록 설계하였다. 다음 그림 1은 윗면의 구성을 보여줍니다.

![윗면 구성|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/b40b82f60004.webp)

<center>그림 1 자판기 윗면 구성</center>

출력부에서는 360도 서보모터를 이용하여 물건이 출력할 수 있게 하였습니다. 모터가 회전하면 내부의 철사가 회전하여 상품이 출력됩니다. 이는 그림 2를 통해 확인할 수 있습니다.

![윗면 구성 (2)|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/bf0e8fa461ef.webp){: width="30%" height="30%"}

<center>그림 2 상품 출력의 원리</center>

### 3) 전면 구성

그림 3과 그림 4는 자판기의 전면의 사진입니다. 전면에는 LCD 모듈, 버튼 3개, 적외선 감지 모듈, LED 2개, 동전 투입부과 상품 출력부, 동전 반환부가 위치하고 있습니다. 각자의 기능은 다음과 같습니다.

![전면 구성|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/a5413c896f2c.webp)

<center>그림 3 자판기 전면 구성 1</center>

![전면 구성 (2)|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/8326c3f2170c.webp){: width="500px" height=""}

<center>그림 4 자판기 전면 구성 2</center>

`LCD 모듈`
LCD 모듈은 2줄의 문자열을 출력해주는 장치입니다. 이번 프로젝트에서 사용한 모듈은 LCD 1602 I2C 모듈로, 기존 LCD 모듈에서 GPIO핀을 많이 사용한 문제를 I2C 프로토콜을 사용해 해결하였습니다. I2C 프로토콜은 SDA (Serial Data)와 SCL (Serial Clock) 두 개의 선호선으로 구성되어 있습니다. 최대 127개의 장치를 단 두 개의 신호선으로 연결할 수 있는 것이 장점이고, 최대속도는 400kHz입니다.

이번 프로젝트에서 LCD 모듈은 자판기의 상태를 보여주는 역할을 합니다. 예를 들어, 넣은 동전의 금액, 구매가 완료되었는지 여부 등을 알려줍니다.

`구매/반환/호출 버튼`
각 버튼은 사용자가 임의로 누르는 버튼으로, 구매하고 싶다면 구매 버튼, 넣은 동전을 반환하고 싶다면 반환 버튼, 자판기에 문제가 발생하여 관리자를 호출하고 싶다면 관리자 호출 버튼을 누르게 됩니다.

`적외선 감지 센서 모듈`
자판기에는 총 3개의 TCRT5000 적외선 감지 센서 모듈이 포함됩니다. 그중 하나의 모듈이 전면의 상품 출력부에 사용됩니다. 그림 5는 TCRT5000 적외선 감지 센서 모듈의 스팩을 나타내고 있습니다. 아날로그 출력으로 사용 시 적외선을 통해 거리를 측정할 수 있으며, 프로젝트에서 사용된 것과 같이 디지털 출력으로 사용 시에는 센서 앞에 사물이 있는지 없는지 확인하는 용도로 사용할 수 있습니다.

![전면 구성 (3)|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/bc321bad441d.webp)

<center>그림 5 TCRT5000 적외선 감지 센서 모듈의 스팩</center>

상품의 출력부에 쓰인 적외선 감지 모듈은 2가지 역할을 수행합니다. 하나는 상품이 출력될 때, 360도 서보모터가 회전하는 데, 상품이 완전히 출력된 후에 서보모터를 정지시켜주는 역할을 합니다. 이때, 상품이 출력되었는지 감지합니다. 또 다른 역할은 도난 방지 기능을 수행합니다. 상품이 출력되고 있지 않을 때, 출력부에 손을 넣어 물품을 도난 하려고 하는 것을 감지합니다.

`상품 출력 LED와 경고 LED`
상품 출력 LED는 상품이 정상적으로 구매되어 상품이 출력되고 있음을 눈으로 확인할 수 있는 역할을 수행합니다. 그리고 경고 LED는 상품의 도난이 의심되었을 때, 이용자에게 경고하기 위해 사용됩니다.

`동전 투입부, 동전 반환부, 상품 출력부`
동전을 투입하는 곳, 동전을 반환하는 곳, 상품을 출력하는 곳입니다.

### 4) 좌측면 구성

그림 6, 그림 7은 자판기의 좌측면을 보여주는 사진입니다. 문을 열릴 수 있도록 설계하여, 동전을 회수할 수 있고 유지관리가 가능하게 하였습니다. 또한, 경고 버저가 위치해, 사용자에게 도난 경고를 할 수 있도록 하였습니다.

좌측면은 크게 동전 분리부와 동전 선택부로 구분할 수 있습니다. 각자의 기능은 다음과 같습니다.

![좌측면 구성|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/3d83d6ea10a2.webp)

<center>그림 6 자판기 좌측면 구성 1</center>

![좌측면 구성 (2)|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/66de282ca97b.webp){: width="500px" height=""}

<center>그림 7 자판기 좌측면 구성 2</center>

`동전 분리부`
동전 분리부는 동전 투입부로부터 들어온 500원, 100원을 분류할 수 있는 부분입니다. 원리는 그림 8을 보면 알 수 있습니다. 동전 투입부로부터 들어온 동전은 레일을 타고 내려오게 됩니다. 이때, 500원과 1000원은 가속도가 붙게 됩니다. 가속도가 붙은 100원은 500원짜리가 떨어질 수 있는 구멍을 지나치게 되어 100원 센서 밑으로 떨어지게 됩니다. 반면에 500원은 보라색 부분의 조형물에 부딪혀 속도를 잃고 500원 센서 밑으로 떨어지게 됩니다.

![좌측면 구성 (3)|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/97b2677334f0.webp){: width="500px" height=""}

<center>그림 8 동전 분류의 원리</center>

`동전 선택부`
위 동전 분리부로부터 분류된 동전들은 센서를 지나 동전 선택부에 모이게 됩니다. 그림 9는 동전 보관과 반환의 원리에 관해서 설명하고 있습니다. 동전 선택부에는 왼쪽과 오른쪽, 가운데를 선택할 수 있는 판막이 하나가 존재합니다. 이 판막은 180도 서보모터를 통해 구현하였습니다. 180도 서보모터는 “6)중간면”에 위치하고 있습니다. 서보 모터를 조정하여 판막이 왼쪽으로 쏠리게 되면 동전은 자판기 내부로 들어가게 되고, 오른쪽으로 쏠리게 된다면 동전은 동전 반환부로 반환되게 됩니다.

![좌측면 구성 (4)|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/9e8cd8dcd5e0.webp){: width="500px" height=""}

<center>그림 9 동전 보관과 반환의 원리</center>

### 5) 우측면 구성
그림 10은 자판기의 우측면을 보여줍니다. 자판기의 모든 핀을 제어하기 위해 라즈베리파이와 연결해 주었습니다.

![우측면 구성|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/575e3c8f8775.webp){: width="500px" height=""}

<center>그림 10 자판기 우측면 구성</center>

5V를 사용하는 부품과 3.3V를 사용하는 부품을 나누어 연결해 주었습니다. 이를 위해서 5V는 외부 전원을 통해 인가하였고, 3.3V는 라즈베리파이 내부의 전원을 활용하였습니다. 또한, 손쉬운 연결을 위해 라즈베리파이 GPIO T 보드와 빵판을 사용하였습니다. 그림 11은 완성된 회로 구성입니다.

### 6) 후면 구성
후면은 그림 12과 같이 구성하였습니다.

![후면 구성|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/fa7dab51ee63.webp){: width="500px" height=""}

<center>그림 12 자판기 후면 구성</center>

### 7) 중간면 구성
중간면은 그림 13, 그림 14와 같이 구성하였습니다. 자석을 이용하여 system과 출력부를 분리할 수 있도록 하여 유지보수에 할 수 이도록 설계하였습니다. 중간 면에는 적외선 감지 센서 2개와 180도 서보모터가 위치해있습니다. 각자의 기능은 다음과 같습니다.

![중간면 구성|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/9112c6a545ed.webp){: width="500px" height=""}

<center>그림 13 자판기 중간면 구성 1</center>

![중간면 구성 (2)|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/81a9417938ea.webp){: width="500px" height=""}

<center>그림 14 자판기 중간면 구성 2</center>

`적외선 감지 센서 모듈`
자판기에는 총 3개의 적외선 감지 센서 모듈 중 두 개의 모듈이 중간면에 사용됩니다. 각각의 센서는 “4) 좌측면”에서 분류된 100원과 500원을 감지해 개수를 세주는 역할을 합니다.

`180도 서보모터`
“4) 좌측면”에서 동전 선택부에 모인 동전을 보관할지, 반환할지 결정하는 역할을 수행합니다. 모터의 각도에 따라 판막이 왼쪽으로 쏠리게 되면 동전은 자판기 내부로 들어가게 되고, 오른쪽으로 쏠리게 된다면 동전은 동전 반환부로 반환되게 됩니다.

## 3. SW 구성
SW 설계는 파이썬을 통해 설계를 진행하였습니다. 파이썬을 선택한 이유는 파이어베이스 데이터 서비스를 이용하기 위해서입니다. 파이어베이스에서는 여러 언어를 지원하는데, 그 중 하나가 파이썬이고 아쉽게도 C를 지원하지 않습니다. 물론 JSON 규격을 이용하기에 규격을 주고받을 수 있는 함수를 작성하여 실행한다면 구현할 수 있겠지만, 이미 라이브러리를 제공해주는 파이썬을 사용하는 것이 시간 측면에서 큰 이점으로 다가와 파이썬을 선택하였습니다.

외부 라이브러리는 python에서 제공해주는 time 모듈, 구글의 firebase를 사용하기 위한 모듈, GPIO컨트롤을 위한 모듈, LCD화면을 출력하기 위해 I2C 통신을 사용하는 모듈을 사용하였다. 외부 라이브러리를 제외한 모든 코드는 직접 구현하였습니다.

외부 센서는 기본적으로 인터럽트 처리를 통해 시스템의 전력을 줄이고 쓸데없는 CPU Time을 줄였습니다. 메인 함수는 Polling으로 작성하여 시스템이 계속 수행되도록 하였습니다.

### 1) 외부 라이브러리 import

![외부 라이브러리 import|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/2b29007accf5.webp){: width="300px" height=""}

<center>그림 15 외부 라이브러리 import</center>

`time 모듈`
외부 라이브러리로 그림 15와 같이 import를 해주었습니다. time은 sleep() 함수를 사용하기 위해 import를 해주었습니다. sleep() 함수는 일정 시간 동안 delay를 줄 수 있는 함수입니다.

`firebase 모듈`
파이어베이스는 구글에서 제공하는 온라인 데이터베이스입니다. 실시간 데이터를 지원하므로 데이터를 실시간으로 주고받을 수 있습니다. 파이어 베이스 모듈은 그림 16과 같이 구성됩니다.

![외부 라이브러리 import (2)|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/6e8a73759e30.webp)

<center>그림 16 firebase 모듈 구성</center>

![외부 라이브러리 import (3)|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/0d4752c28728.webp)

<center>그림 17 firebase setting</center>

먼저 firebase 모듈 일부분인 `credentials`로부터 인증 정보를 가져와야 합니다. 그 이유는 firebase 데이터베이스 상에 아무나 접근을 하면 보안상의 문제가 있기 때문입니다. `Cerificate()` 메서드는 그림 18과 같이 정의되어 있습니다. 읽어보면 <u>JSON 파일로부터 인증 정보를 가져온다</u>고 되어있습니다.

> JSON은 속성-값의 쌍으로 데이터를 주고받을 수 있는 포맷입니다. 이 JSON 인증 파일은 구글 firebase로부터 다운로드할 수 있으며, 이용자마다 모두 다릅니다.

이 파일을 라즈베리파이의 `/home/pi/Project` 디렉터리 밑에 넣어주었고 함수의 인자로 전달해주면 인증된 값을 받을 수 있습니다. `cred` 변수에 이 인증값을 저장하였습니다.

![외부 라이브러리 import (4)|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/009ed70f73b2.webp){: width="500px" height=""}

<center>그림 18 Cerificate() 메서드</center>

이렇게 구한 `cred`는 firebase를 초기화해주는 데 이용합니다. 그림 19는 firebase를 초기화해주기 위한 `initialize_app()`함수에 대한 정의입니다. `cred`와 `databaseURL`을 받아 firebase를 초기화해줍니다. `databaseURL`은 firebase상에서 <u>데이터가 저장되어 있는 위치</u>입니다. firebase 데이터 베이스 상에서도 많은 정보가 있기에, 원하는 위치를 특정지어 주기 위해서 위와 같이 설정을 합니다.

![외부 라이브러리 import (5)|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/1bd53fdfb455.webp){: width="500px" height=""}

<center>그림 19 initialize_app() 메서드</center>

마지막 단계로 `ref`를 받아와야 합니다. 그림 20은 firebase에서 reference를 받아오기 위한 `db.reference()`함수에 대한 정의입니다. 모든 인자들은 선택적으로 받아올 수 있고, 받아오지 않았을 경우 위에서 선택한 `database URL`로 선택됩니다.

![외부 라이브러리 import (6)|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/1425cb3b14a3.webp){: width="500px" height=""}

<center>그림 20 db.reference() 메서드</center>

결과적으로 <u>firebase로부터의 연동을 완료</u>하였고, reference를 받아와서 실시간 데이터베이스에 접근할 수 있게 되었습니다. 그림 21은 firebase 상의 데이터 베이스이고 라즈베리파이와 연동이 되었습니다.

데이터 베이스는 `vendingM`밑에 `Message`, `RaspUpdateSignal`, `earnCoin`, `stock`이 존재한다. 각자의 기능은 다음과 같습니다.

`Message`: 라즈베리파이로부터 받는 메시지로 관리자가 꼭 알아야 하는 문구를 표시할 수 있습니다.

`RaspUpdateSignal`: Firebase로부터 라즈베리파이의 데이터를 갱신할 때 사용되는 신호입니다.

`earnCoin`: 현재까지 번 돈의 금액을 표시해줍니다.

`stock`: 자판기의 재고를 표시해줍니다.

![외부 라이브러리 import (7)|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/7ae2526663d8.webp)

<center>그림 21 파이어 베이스 실시간 데이터 베이스</center>

`LCD 모듈`
LCD를 I2C 통신으로 제어하기 위해 import 해주었습니다.

`GPIO 모듈`
GPIO를 제어하기 위해 import 해주었습니다.

### 2) 핀 설정

![핀 설정|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/82a21d0482fb.webp){: width="400px" height=""}

<center>그림 22 핀 설정</center>

그림 22는 system 상에서 사용하는 모든 Pin을 정의한 코드입니다.

그림 23는 정의한 핀들을 GPIO 설정을 통해 input, out mode를 설정하고, pull-up, down mode를 지정해주는 코드입니다.

(https://github.com/radic2682/radic2682.github.io/assets/11177959/3a9c065d-93c6-4439-8a4c-8e576b6c6b7d)

<center>그림 23 핀과 GPIO 설정</center>

그림 24를 보면 `GPIO.setup()`함수에서 <u>pull-up, down mode</u>에 대한 설명이 있습니다. 라즈베리파이 내부의 풀업, 다운 저항을 사용하여 설정함으로써 회로를 직접 구현하지 않아도 된다는 장점이 있습니다. 각 상태에 대한 정의는 그림 25를 보면 확인할 수 있습니다.

![핀 설정 (2)|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/b00d172be8b1.webp)

<center>그림 24 내장 저항 사용</center>

![핀 설정 (3)|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/d6911cfe1020.webp){: width="500px" height=""}

<center>그림 25 풀 업 저항과 풀 다운 저항이란</center>

### 3) 모터 설정

![모터 설정|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/6f2e3c09fe31.webp){: width="500px" height=""}

<center>그림 24 모터 설정</center>

라즈베리파이에서 모터는 `PWM` 신호로 제어할 수 있습니다. `PWM` 제어 방식은 사각파의 폭을 제어한다는 의미입니다. PWM 제어에서는 크게 주파수와 Duty ratio라는 두 가지 파라미터가 있습니다. `주파수`는 1초에 진동하는 횟수이며, `Duty ratio`는 한 주기 안에서 신호가 on 되어있는 비율입니다. 그림 25는 `PWM` 제어 방식을 표현하는 그림입니다.

![모터 설정 (2)|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/34f44399633e.webp)

<center>그림 25 PWM 제어 방식</center>

그림 24 코드를 보면 180도 모터의 Duty 값을 정의해두었습니다. 이는 실험을 통해 얻은 값이며 11.5에서는 왼쪽으로, 9.0에서는 중앙, 7.0에서는 오른쪽으로 회전하는 값입니다. 그림 26은 Duty 값별 움직임을 나타냅니다.

![모터 설정 (3)|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/98a356e99a7a.webp)

<center>그림 26 180도 모터의 Duty 별 움직임</center>

서보모터가 PWM 동작을 하기 위해 50MHz 주파수를 인자로 전달해주어 PWM 360, PWM 180개체를 선언해 주었습니다. 각각 360도 서보모터와 180도 서보모터를 나타내는 변수입니다. 그 후 개체의 start() 메서드를 실행하여 초기 상태를 지정해주었습니다.

360도 서보모터의 주기별 작동은 그림 27과 같습니다. 360도 모터는 180도 모터와 다르게 0에서는 정지하고 1.5 주기에서는 계속해서 회전한다는 특징이 있습니다.

![모터 설정 (4)|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/04cf875fa265.webp)

<center>그림 27 360도 모터의 Duty 별 움직임</center>

ChangeDutyCycle()은 PWM의 Duty를 변경시키는 메서드입니다. 물론 초기화를 같은 값으로 해주긴 하였지만, 모터가 이동할 시간을 주지 않았었습니다. 그렇기에 확실하게 하기 위해 ChangeDutyCycle()를 이용하여 모터를 가운데로 설정하였습니다. time.sleep()은 delay함수로 모터가 이동할 시간을 줍니다.

### 4) VendingMachine 클래스 정의

VendingMachine class는 자판기의 기본적인 변수와 함수를 제공하기 위해 만들었습니다. 만약 관리해야 하는 자판기의 개수가 늘어난다면 코드가 굉장히 길어질 텐데, 이때 코드의 재사용과 관리의 편의성을 위해 만들었습니다.

![VendingMachine 클래스 정의|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/44406003b44f.webp)

<center>그림 28 VendingMachine Class</center>

![VendingMachine 클래스 정의 (2)|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/6a51277711cb.webp)

<center>그림 29 VendingMachine 클래스 다이어그램</center>

그림 29는 클래스 다이어그램으로 나타낸 VendingMachine 클래스입니다. 기본적으로 5개의 변수를 내장하고 있습니다. 각 변수의 기능은 다음과 같습니다.

`count_500` : 자판기에 들어온 500원의 개수

`count_100` : 자판기에 들어온 100원의 개수

`count_earn` : 현재까지 수입

`count_stock` : 자판기에 남아 있는 재고의 수량

`product_output_ongoing` : 자판기에서 상품을 출력 중인지 확인하는 변수

이러한 변수들은 VendingMachine 클래스의 개체가 만들어지면서 초기화되는데, 이는 그림 28의 `__init__`부분을 보면 알 수 있습니다.

VendingMachine 클래스는 7개의 메서드를 가지고 있습니다. 각 메서드의 기능은 다음과 같습니다.

`Reset()` : 자판기에 들어온 100원, 500원의 개수를 초기화하는 메서드

`Calculate()` : 현재 자판기에 들어온 금액을 계산해 return 하는 메서드

`DisplaySetting()` : LCD 두 번째 줄에 들어온 금액을 보여주는 메서드

`Add(coin)` : 자판기에 들어온 100원, 500원의 개수를 추가하는 메서드

`AccumulateCoin()` : 자판기의 수입을 추가해주는 메서드 (가격이 700원이므로 += 700)

`ReduceStock()` : 재고를 하나 줄여주는 메서드

`updateFromDatabase()` : 데이터베이스로부터 값을 읽어와 변수를 업데이트해주는 메서드

`updateFromDatabase()` 메서드는 데이터베이스의 vendingM의 `stock`과 earnCoin의 `value` 값을 읽어와 count_stock과 count_earn을 <u>갱신해주는 역할을 수행</u>합니다. 받아올 때, `db.reference()`의 `getter`를 통해 값을 받아 int형으로 변환합니다. 외부 데이터베이스와 통신을 하기 때문에 <u>예외 처리</u>를 해주어야 합니다. try-catch문을 이용해 exception을 발생시켜서, 통신이 잘되지 않았을 때 시스템이 fault되는 것을 방지했습니다.

### 5) 디스플레이 개체 생성

![디스플레이 개체 생성|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/712efc036f4a.webp)

<center>그림 30 디스플레이 개체 생성</center>

외부 라이브러리인 I2C의 LCD 클래스의 개체를 생성하는 코드입니다. LCD 클래스는 LCD를 `I2C` 통신을 통해 값을 전달해주는 역할을 합니다. 프로젝트에서는 LCD 클래스에서 하나의 메서드만 사용합니다. 그림 31은 프로젝트에서 사용하는 `LCD-display_string()` 메서드에 대한 정의입니다. 선택해준 line에 따라 문자열을 출력할 수 있게 되어있으며 문자열을 foreach문을 이용해 한 문자씩 전달해 출력하는 것 또한 확인할 수 있었습니다. 더 자세한 코드는 라이브러리를 열어 봄으로써 확인할 수 있습니다.

![디스플레이 개체 생성 (2)|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/91f3e72f0708.webp){: width="400px" height=""}

<center>그림 31 LCD 클래스의 LCD-display_string 메서드</center>

### 6) buttonPushed(sentence, m_control1), initDisplay() 함수 생성

![buttonPushed(sentence, m_control1), initDisplay() 함수 생성|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/18f481c7b6ec.webp){: width="400px" height=""}

<center>그림 32 buttonPushed(sentence, m_control1), initDisplay() 함수</center>

`buttonPushed()` 함수는 sentence, m_control1을 인자로 받아 LCD의 두 번째 줄에 sentence를 출력해주고 180도 모터의 위치를 m_control1 주기에 해당하는 값으로 변경한 후 다시 중앙으로 이동해주는 역할을 수행하는 함수입니다.

`time.sleep()`은 모터가 이동할 시간을 줍니다.

`initDisplay()` 함수는 초기 디스플레이값을 설정해주는 함수로 시스템이 어떠한 역할도 수행하지 않을 때 표시해줄 값입다.

### 7) 초기화

![초기화|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/1a0344fd3a11.webp)

<center>그림 33 초기화</center>

멤버 변수를 전부 0, False값으로 하는 `VendingMachine` 클래스의 개체를 생성합니다. 그 후 `vendingMachine`의 메서드 `updateFromDatabase()`를 실행시켜 데이터베이스로부터 값을 읽어와 변수를 초기화해줍니다.

`initDisplay()`를 호출해 초기 디스플레이값을 설정해줍니다.

### 8) 인터럽트 처리 함수 - Coin 관련

![인터럽트 처리 함수 - Coin 관련|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/9fe0734ffaa6.webp){: width="500px" height=""}

<center>그림 34 Coin 관련 인터럽트</center>

그림 34와 같은 Coin의 개수를 세줘야 하는 인터럽트 발생 시 다음과 그림 35와 같은 함수를 수행합니다. 인터럽트는 적외선 센서로부터 발생합니다.

![인터럽트 처리 함수 - Coin 관련 (2)|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/3ed874048aa8.webp){: width="500px" height=""}
![인터럽트 처리 함수 - Coin 관련 (3)|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/aa7a1f0516b1.webp){: width="500px" height=""}

<center>그림 35 callback_By_500(channel), callback_By_100(channel)</center>

함수는 각각 500원과 100원의 개수를 추가하고 자판기의 LCD에 들어온 동전의 총금액을 표기해줍니다.

### 9) 인터럽트 처리 함수 - return 관련

![인터럽트 처리 함수 - return 관련|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/df1d68ea5e30.webp){: width="300px" height=""}

<center>그림 36 return 관련 인터럽트</center>

그림 36와 같은 동전을 반환해줘야 하는 인터럽트 발생 시 다음과 그림 37와 같은 함수를 수행합니다. 인터럽트는 버튼으로부터 발생합니다.

우선 자판기에 현재까지 넣은 동전을 초기화해줍니다. 그 후 반환 완료 메시지를 LCD에 출력 후, 180도 모터를 반환 방향으로 회전하여 동전을 반환 후 디스플레이를 초기화해줍니다.

![인터럽트 처리 함수 - return 관련 (2)|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/a94e1024805b.webp){: width="500px" height=""}



<center>그림 37 callback_By_return_btn_pin(channel)</center>

### 10) 인터럽트 처리 함수 - purchase관련

![인터럽트 처리 함수 - purchase관련|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/9661a38eb4b7.webp){: width="300px" height=""}

<center>그림 38 purchase관련 인터럽트</center>

그림 38와 구매 관련하여 처리를 해줘야 하는 인터럽트 발생 시 다음과 그림 39와 같은 함수를 수행합니다. 인터럽트는 버튼으로부터 발생합니다.

![인터럽트 처리 함수 - purchase관련 (2)|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/de43a5ad58d0.webp)

<center>그림 39 callback_By_purchase_btn_pin(channel)</center>

구매 관련 인터럽트를 플로우 차트로 그려서 파트를 나누면 그림 40과 같이 나타낼 수 있습니다. 아래는 각 파트 별 설명입니다.

![인터럽트 처리 함수 - purchase관련 (3)|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/3702570bc301.webp)

<center>그림 40 callback_By_purchase_btn_pin(channel)의 플로우 차트</center>

#### A

![A|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/b30b1bfce47d.webp)

<center>그림 41 callback_By_purchase_btn_pin(channel)의 플로우 차트의 A</center>

A 파트는 자판기의 재고가 0이 아니고 700원을 사용자가 정확히 넣었을 때 수행됩니다. 즉, 구매가 정상적으로 완료된 상태에서 수행됩니다.

초록색 LED를 켜고 구매 완료 메시지를 LCD에 출력 후, 180도 모터를 동전 보관 방향으로 회전하여 동전을 보관합니다. 그 후, 현재까지 번 돈에 700원을 더하고 재고를 1 감소합니다. 또, 자판기에 현재까지 넣은 동전을 초기화해줍니다.

이렇게 자판기 내부 변수를 모두 업데이트해준 뒤, 데이터를 온라인 데이터 베이스에 업데이트를 해줍니다. 그림 42는 실시간 데이터베이스 상에서 업데이트되는 정보를 보여줍니다.

![A (2)|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/755a0d7c32c0.webp){: width="500px" height=""}

<center>그림 42 실시간 데이터 베이스 업데이트</center>

마지막으로 상품을 출력해 줘야 하는데, `product_output_ongoing` 변수를 True로 지정하여 출력되고 있음을 지정해줘야 합니다. 그 이유는 `product_output_ongoing`이 True가 아닌 상태에서 상품 출력부에 손을 넣으면 도난 경보가 울리기 때문입니다. 360도 모터의 주기를 1.5로 설정해 회전시키며 인터럽트 함수처리를 마칩니다. 360도 모터의 회전 중단의 처리는 출력부의 적외선 센서의 인터럽트로 처리하며 이 부분은 “12) 인터럽트 처리 함수 – 모터 정지 및 도난 방지 관련”에 설명하도록 합니다.

#### B

![B|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/abd5ac1779da.webp)

<center>그림 43 callback_By_purchase_btn_pin(channel)의 플로우 차트의 B</center>

B 파트는 자판기의 재고가 0이 아니고 사용자가 700원보다 적게 돈을 넣었을 때 수행됩니다. 이러한 경우, 돈을 적게 넣었음을 사용자에게 알려주고 지금까지 넣어준 돈을 표시해줘야 합다.

#### C

![C|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/789000030458.webp)

그림 44 callback_By_purchase_btn_pin(channel)의 플로우 차트의 C

C 파트는 자판기의 재고가 0이 아니고 사용자가 700원보다 더 돈을 넣었을 때 수행됩니다. 이러한 경우, 돈을 많이 넣었음을 사용자에게 알려주고 지금까지 넣어준 돈을 반환하고 자판기가 초기 상태가 됩니다.

#### D

![D|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/a5e46a681cb3.webp)

<center>그림 45 callback_By_purchase_btn_pin(channel)의 플로우 차트의 D</center>

D 파트는 자판기의 재고가 없을 때 수행됩니다. 이러한 경우, 재고가 없음을 사용자에게 알려주고 지금까지 넣어준 돈을 반환하고 자판기가 초기 상태가 됩니다.

### 11) 인터럽트 처리 함수 – 관리자 호출 관련

![인터럽트 처리 함수 – 관리자 호출 관련|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/db7323b84a21.webp){: width="300px" height=""}

<center>그림 46 관리자 호출 관련 인터럽트</center>

그림 46와 관리자 호출 관련하여 처리를 해줘야 하는 인터럽트 발생 시 다음과 그림 47과 같은 함수를 수행합니다. 인터럽트는 버튼으로부터 발생합다.

![인터럽트 처리 함수 – 관리자 호출 관련 (2)|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/6464fe098520.webp)

<center>그림 47 callback_By_call_btn_pin(channel)</center>

`callback` 인터럽트 함수는 사용자에게 문제가 생겨 관리자를 호출하는 버튼을 눌렀을 때 발생합니다. 그림 48과 같이 관리자에게 자판기에 문제가 발생했음을 알리는 문구를 데이터 베이스 상에 업데이트해준 뒤, 사용자에게 업데이트가 성공적으로 완료되었음을 LCD로 알려준 뒤 LCD를 초기화합니다.

![인터럽트 처리 함수 – 관리자 호출 관련 (3)|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/09607f700ccb.webp){: width="300px" height=""}

<center>그림 48 실시간 데이터 (관리자 호출)</center>

#### 12) 인터럽트 처리 함수 – 모터 정지 및 도난 방지 관련

![인터럽트 처리 함수 – 모터 정지 및 도난 방지 관련|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/99a8b632f48d.webp){: width="500px" height=""}

<center>그림 49 모터 정지 및 도난 방지 인터럽트</center>

그림 49와 모터를 정지하고 도난을 방지하는 기능에 관련하여 처리를 해줘야 하는 인터럽트 발생 시 다음과 그림 50과 같은 함수를 수행합니다. 인터럽트는 버튼으로부터 발생합니다.

![인터럽트 처리 함수 – 모터 정지 및 도난 방지 관련 (2)|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/c235a0c4386e.webp)

<center>그림 50 callback_By_sensorStopMotorandWarning_btn_pin(channel)</center>

`callback_By_sensorStopMotorandWarning_btn_pin(channel)` 인터럽트 처리 함수는 크게 두 가지 기능을 수행합니다. 하나는 `product_output_ongoing` 변수가 True일 때(상품이 출력중 일 때)이고 나머지 하나는 `product_output_ongoing` 변수가 False일 때(상품이 출력되고 있지 않을 때)이다. 각 상황에 따라 다음과 같이 처리합니다.

`product_output_ongoing = True (상품이 출력중 일 때)`

상품이 출력부로 출력이 되면 적외선 센서에 의해 감지가 될 것이고, 이때 서보모터를 정지해줘야 한다. 그 후, 설정했던 초록 LED를 끄고, `product_output_ongoing` 변수를 다시 False로 만들어 상품이 출력되고 있지 않음을 설정해준다.

`product_output_ongoing = False (상품이 출력되고 있지 않을 때)`

상품이 출력되고 있지 않을 때, 출력부에 손을 넣으면 도난 경보가 울려야 한다. 이를 위해 부저와 빨간 LED에 불이 1초 동안 들어오도록 설계하였다. 그 후, 그림 51과 같이 관리자에게 자판기에 도난 문제가 발생했음을 알리는 문구를 데이터 베이스 상에 업데이트해준다.

![인터럽트 처리 함수 – 모터 정지 및 도난 방지 관련 (3)|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/3a0294a5447b.webp){: width="300px" height=""}

<center>그림 51 실시간 데이터 (모터 정지 및 도난 방지)</center>

### 13) 인터럽트 설정

![인터럽트 설정|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/dffee9bd7458.webp)

<center>그림 52 인터럽트 설정</center>

8)~12)까지 인터럽트 발생 시 처리하는 `subroutine` 함수를 작성했다. 이 함수들을 시스템에서 제공하는 함수를 이용하여 핀과 연결해주는 작업이 필요합니다. 그림 53을 보면 add_event_detect 함수의 정의를 확인할 수 있습니다. 인자로 핀번호, 상태설정, subroutine 함수, bouncetime을 받습니다. 상태는 어떠한 Egde에서 인터럽트를 감지하는지에 대해서 설정할 수 있습니다. 처음에 GPIO 핀을 설정할 때, 풀 업 저항을 이용하여 High를 default 상태로 두었기 때문에 Falling Edge에서 감지하도록 설정을 하였습니다.

`bouncetime`은 인터럽트 처리 함수를 두 번이상 호출되는 것을 막기 위해서 설정하였고, 적절한 값을 찾아 넣어주었습니다.

![인터럽트 설정 (2)|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/95ab7d753784.webp)

<center>그림 53 인터럽트 Edge</center>

### 14) main 함수 구현

![main 함수 구현|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/9153ce9dd2a2.webp)

<center>그림 54 Main</center>

그림 54는 메인 함수이다. 파이썬은 위에서 아래로 순차적으로 수행하면서 실행됩니다. 즉, 필요한 함수를 모두 선언한 뒤 `Main` 문을 작성하였으며 메인 함수의 이름은 딱히 지정하지 않았습니다.

try-catch문을 이용하여 예외를 처리하였습니다. 온라인으로 통신을 하는 것이기에 예외처리는 필수적입니다. 만약에 서버의 통신으로부터 값을 제대로 받아오지 못한 경우, 예외 처리가 없다면 system이 fault가 나기 때문입니다. 또한, 무한 반복문을 빠져나오기 위해서 Ctrl + C를 입력했을 때 exception 처리를 해주었습니다.

메인문은 기본적으로 대기하는 기능을 주 기능으로 가지고 있으며, 추가적으로 그림 55와 같이 `RaspUpdateSignal`이 1이면 파이어베이스 데이터 베이스로부터 라즈베리파이로 데이터를 업데이트해주는 기능을 수행합니다. 그렇기 위해서는 매번 파이어베이스로부터 `RaspUpdateSignal`이 1인지 확인할 필요가 있습니다. 모든 업데이트가 마치면, 라즈베리파이에서 파이어베이스로 `RaspUpdateSignal`을 0으로 초기화해줍니다.

![main 함수 구현 (2)|medium](/img/01HG4ET100AYM7JVXNQB4X62XV/3840ea9c089f.webp)

<center>그림 55 자판기 작동 중, 파이어베이스로부터 정보 업데이트 매커니즘</center>

마지막으로 `finally`문을 이용하여 자원을 반환합니다. 이는 프로그램을 안전하게 종료시키는 목적을 가지며, 다음번에 프로그램을 재실행했을 때 문제가 발생하지 않도록 합니다.

## 4. 고찰

임베디드 기말 프로젝트로 IOT를 활용한 자판기를 설계하였습니다. 자판기로써의 기본적인 기능을 함과 동시에 관리자가 자판기를 관리하기 쉽게 하는 여러 기능을 추가하였습니다. 이 과정에서 여러 센서와 라즈베리파이를 사용하였고, 프로그래밍 언어는 파이썬을 사용하였습니다. 기본적인 구현은 인터럽트를 통하여 시스템의 성능을 개선시켰습니다.

프로젝트를 수행하며 2학기에 배운 인터럽트와 기본적인 임베디드 시스템에 대한 이해가 높아지는 계기가 될 수 있었습니다.

(Note) 이 프로젝트에 관한 모든 코드는 GitHub에 공유되어 있습니다.

[놀러가기](https://github.com/radic2682/IOT_Vending_Machine/blob/main/mainModify4.py)

(Note) 이 프로젝트에 관한 영상이 있습니다.

[놀러가기](https://youtu.be/HnQytw9c-0U?si=cZ31_LSgpW0Xq_FK)

---

<center><u>Sun Hong</u></center>
