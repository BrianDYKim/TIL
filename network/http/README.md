## 1. HTTP/0.9
HTTP/0.9는 아래의 특징을 가진다.

* 단 하나의 HTTP 메소드를 가진다 (GET)
* HTTP 헤더가 존재하지 않는다
* HTML만 body로 가질수있다.

## 2. HTTP/1.0

HTTP/1.0에 들어서 다음의 내용이 추가되었디.

* HTTP 헤더
* Status code (상태코드)
* Redirect
* Errors
* 조건부요청
* 콘텐츠 인코딩
* 더 다양한 요청 메소드

하지만 이러한 HTTP/1.0도 개선해야할 점은 많은데, 다음과 같다.

* HTTP/1.0은 아직도 1 요청 당 1 TCP Connection을 소비한다. 즉, 1개의 요청이 들어오면 TCP 3-way handshaking을 맺고, request를 처리하고, 4-way handshaking 과정을 거쳐서 다시 연결을 끊어야한다.
handshaking 과정은 많은 오버헤드를 유발하기 때문에 이는 개선해야할 사항이다.
* HOST가 헤더의 필수 사항이 아니었다. RESTful한 API를 지향하기 위해서는 어느 정도 Self-descriptive를 지켜줄 필요가 있는데, HTTP/1.0 까지는 그러하지 못했다

## 3. HTTP/1.1

HTTP/1.1에 들어서 아래의 사항이 지원되기 시작했다.

* 1개의 TCP 커넥션 당 1개 이상의 요청을 처리할 수 있게되었다. HTTP 계층에서 TCP 계층에서 사용하던 timeout 개념을 도입하여 TCP 커넥션을 재활용할 수 있게 되었다. 
즉, Handshaking 과정에서 발생하는 오버헤드가 줄어들었다.
* **HTTP Pipelining이 지원되기 시작하였다.**

하지만 HTTP/1.1의 파이프라이닝은 문제점이 많다. 

* Client 측에서 Pipelining 기법을 이용해서 request를 server에 여러개 보내더라도 서버는 이를 순차처리해서 client로 보내야한다.
* Server는 결국 Pipelining된 Data를 순차처리 해야하기 때문에 앞의 요청이 블로킹 되면 뒤의 요청에도 레이턴시가 걸리게된다. 이를 **HOL(Head Of Line) 블로킹 현상** 이라고 부른다.

따라서 HTTP/1.1 환경에서는 파이프라이닝이 지원이 되더라도 정작 파이프라이닝을 제대로 구현해서 사용하는 곳은 없다고 생각하면된다.

결국 HTTP/2.0에 들어서 해결해야할 문제는 아래의 것들이 있다고 보면된다.

* HTTP/1.1의 고질적 문제인 HOL 블로킹 현상을 해결한다
* 파이프라이닝을 흉내내기 위한 **Client측에서 Server와 다중 TCP Connection을 맺는 문제를 해결해야한다.** 이는 디바이스와 서버 측에 큰 부담을 준다.
* TCP의 Congestion Control와 관련된 동작을 개선해야한다

## 4. HTTP 성능에 영향을 미치는 지표

**1. 네트워크 레벨**
* 지연시간(latency): In-flight 패킷이 한 쪽에서 다른 쪽으로 이동하는데 걸리는 시간. RTT(Round Trip Time)은 이러한 latency의 2배 값이라 생각하면 된다.
* 대역폭(Bandwidth): 두 지점 사이의 연결은 포화 상태가 되기 직전까지의 데이터 양만 통신이 가능하다
* DNS 조회
* 연결 시간: TCP 3-way handshaking time + TLS Handshaking time (optional, you must consider this thing if you use HTTPS Protocol)

![](./img/3-way-handshaking.png)

**2. 그 외의 지표들**
* 바이트 수의 증가: 나날이 갈수록 웹페이지에 띄워야하는 정보가 많아지기 때문에 바이트 수는 나날이 늘어간다
* 개체 수의 증가: 나날이 갈수록 웹페이지에 제공해야하는 개체 수는 증가한다
* 복잡도의 증가
* 호스트 수의 증가: 단일 웹페이지는 다중의 서버와 연결을 맺어서 데이터를 가져온다. 이 과정에서 TCP 3-way handshaking, TLS handshaking 오버헤드가 발생한다
* TCP 소켓 수의 증가: 많은 양의 데이터를 서버로부터 수용받기 위해서 Client는 서버와 다수 개의 TCP connection을 맺고있다. 이는 디바이스의 부담을 더하며, 또한 서버에서 부담이다.

**3. TCP의 Congestion Control 과정이 불러오는 문제**

![](./img/Congestion-Control.png)

위의 그림은 TCP가 Congestion Control을 수행하기 위해서 거치는 과정을 Time-Window 그래프로 표현한 것이다.
HTTP/1.1 프로토콜까지는 파이프라이닝을 사용하지 않기 때문에 Client는 서버와 다중 TCP 커넥션을 맺어야한다.

보통은 Client-Server 간에 6개의 커넥션을 맺는다고 알려져있는데, 그렇게 되면 6개의 커넥션에서 위의 과정이 동일하게 이뤄져야한다.

게다가 아래의 문제가 있다.

* 현대의 웹페이지는 평균 한 번의 응답에 2MB의 개체를 Client로 보내야한다.
* TCP는 하나의 Segment당 1460 bytes로 크기가 한정이 되어있다.
* 이 과정에서 TCP는 대략 9번은 Congestion Control 왕복을 거쳐야한다고 알려져있다.
* 6개의 커넥션을 맺는다면, 6개의 커넥션에서 모두 9번의 Congestion Control 왕복이 발생해버린다.
* 이러한 왕복 과정은 매우 큰 오버헤드를 불러일으킨다.

**4. 매우 비대한 메시지 헤더의 문제**

요청 헤더를 생각해보자. 보통의 요청은 GET으로 수행을 하게될텐데, GET의 경우 HTTP 요청의 대부분을 메시지 헤더가 차지해버린다. 그리고 Client가 브라우저라면 쿠키까지 포함이 되기 때문에, 헤더의 크기는 더욱 비대해진다.

일반적인 웹페이지의 경우 한 요청에 63KB의 요청 헤더가 발생하게 된다. 이를 처리하기 위해서는 TCP Congestion Control이 대략 3~4번의 널뛰기 과정을 거쳐야한다고 알려져있다. 이로 인해서 네트워크 지연이 누적된다.

게다가 Client가 모바일 환경인 경우 Congestion Control이 낮은 지점에서부터 시작을 해야하기 때문에 모바일 환경에서는 더욱 latency가 누적된다.

추가적으로, 포화 상태의 Bandwidth를 사용하는 환경에서는 Bandwidth가 모두 소진이 된 상태에서는 이런 latency가 더욱 잘 누적되는 현상이 벌어진다.

**5. HOL 블로킹이 불러오는 우선순위의 문제**

OS 수업 시간을 떠올려보자. OS의 CPU 스케쥴링은 프로세스마다의 우선순위를 매긴 다음에 프로세스를 효율적으로 관리한다. 필자는 이게 매우 합리적인 현상이라고 생각한다.

그러면 OS의 스케쥴링 기법을 HTTP에도 도입을 하면 되지 않는가 라는 생각을 개발자라면 하게 되어있는데, 안타깝게도 HTTP/1.1은 이런 우선순위를 매겨버리면 큰일난다.

* 우선적으로 처리해야하는 요청을 찾지 못한 경우, 우선적으로 처리해야하는 요청을 HTTP 스트림에서 찾아야하기 때문에 이 과정에서 모든 요청의 처리가 보류된다. 
이 과정에서 레이턴시가 발생한다.
* 찾아도 문제다. 우선순위가 높은 요청을 찾아서 먼저 처리를 수행하면 뒤에 우선순위가 낮은 요청이 밀려버리는 현상이 발생한다. 게다가 해당 요청에서 블로킹이 걸리면 HOL 블로킹 현상이 벌어진다.

## 5. HTTP/2.0을 소개하기 전, HTTP/2.0의 안티패턴부터 소개합니다.

**1. 스프라이팅과 자원 통합/인라이닝**

HTTP/2.0 부터는 파이프라이닝이 지원되기 때문에 하나의 리소스를 쪼개서 다중으로 보낼 필요가 없다. 따라서 스프라이팅은 HTTP/2.0 부터는 안티패턴으로 분류된다.

**2. 샤딩**

위의 내용과 맥락이 같다. 다중 연결을 열 필요가 없기 때문에 요청 샤딩도 필요없다.

**3. 쿠키 없는 도메인**

HTTP/2.0 부터는 파이프라이닝 뿐만이 아니라 헤더 압축(HPACK)도 지원하기 때문에 쿠키를 싣지 않는 요청은 HTTP/2.0부터는 안티패턴이다. 