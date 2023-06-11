# HTTP IN ACTION
## 2. The road to HTTP/2
> 왜 우리는 http/2가 필요할까? http/1.1은 20년동안 잘 작동해 왔지만, 이제 인터넷의 사용자는 폭증했고 웹사이트는 fully interactive해졌다. 속도 역시 빨라졌지만,  더 빠른 속도에 대한 니즈가 훨씬 더 컸다. latency가 가장 중요한 요소인데, 이는 기본적으로 빛의 속도를 넘을 수 없다.

### HTTP/1.1 and the current World Wide Web
http/1.1은 이전에도 다루었다시피, request-response 프로토콜이며 plain-text 컨텐츠 하나를 요청하고 완료되었을 때 연결을 끊는다. 1.0에서 다른 미디어 타입을 도입했고 1.1에서는 커넥션을 끊지 않는게 디폴트가 되었긴 하다.

이러한 개선점들이 좋은건 맞지만, 인터넷은 그보다 훨씬 빠르게 변했다. 평균적으로 하나의 웹 사이트는 80-90개의 리소스를 요청하고 1.8MB의 데이터를 다운로드 한다.  물론, 구글처럼 아주 최적화된 경우는 17개 /0.4MB 정도지만, 웹사이트에서 요청하는 데이터가 늘어나고 있는 트렌드는 확연히 보인다. 

이는 웹 사이트가 media-rich해지고 있고, 여러 프레임워크나 의존성들을 사용해서 점점 더 복잡해지고 있기 때문이다.

웹페이지는 최초에는 정적인 페이지로 시작했지만, 지금은 점점 더 interactive해지면서 서버사이드에서 다이나믹하게 렌더링 되기 시작했다. 그 다음에는 basic html 페이지만 제공받고 ajax 콜로 보충되는, client side의 javascript가 쓰이기 시작했다. 

검색엔진의 예시를 보면 알 수가 있지.

무튼간에, 검색엔진 뿐만 아니라 온갖 종류의 웹페이지들이 ajax 콜을 헤비하게 사용하기 시작했다. 뭐 소셜 미디어나, 이런 것들이. 

결론적으로, website가 이제 web application이 된거다. 그리고, http protocol은 이를 위해서 디자인 된 것도 아니고 근본적인 퍼포먼스 문제가 있다.

### HTTP/1.1’s fundamental performance problem
아주 간단한 웹페이지가 있다고 생각해보자. 
* 두개의 이미지
* 몇몇 텍스트

이제 이 웹페이지를 렌더링 하기 위해 http가 얼마나 비효율적인지 한번 살펴봅시다.

* 요청을 날렸을 때 인터넷을 떠돌며 서버에 도착하는 시간 : 50ms
* static 웹사이트이기 때문에, 서버가 파일에서 페이지를 꺼내 전송까지 걸리는 시간 : 10ms
* 브라우저가 받은 응답을 렌더링 하고 다음 요청을 보낼 때 까지 걸리는 시간 : 10ms

사실 이정도로 간단하지는 않지만, 실례는 있다가 살펴보도록 하고 러프하게  한번 봅시다.

[image:2FBA1779-02FF-46D6-B90C-9A7C9D2BD86A-84333-0000C0261554C908/2918132C-E44C-43EE-A078-B1472A7826D3.png]

여기서 블록이 화살표 하나가 50초, 블록 하나가 10초라고 보면 되겠다. 
즉, 실제로 클라이언트나 서버가 요청을 처리하는데 걸리는 시간은 60초다.  
* 맨 위 블록은 땅 요청을 보내는 시점이고
* 그 아래 6개 블록이 실질적인 처리
* 하지만 **위 그림에서 총 소요시간은 300ms** 
* 즉, 정작 처리하는데는 60ms면 되지만 80%인 300ms가 메시지를 인터넷에 보내고 기다리는데 사용되었다는 것
* 낭비되는 시간

그리고 이 낭비되는 시간이 http 프로토콜의 근본적인 문제다.  그림 하나를 요청하고 나서, 클라이언트는 그림 2가 필요한걸 알지만, 커넥션이 빌 때까지 기다려야 한다.

현대 인터넷의 가장 큰 문제는 bandwith 보다는 /latency/ 이다. 

latency란?
> latency measures how long it takes to send a single message to the server, whereas bandwidth measures how much a user can download in those messages

	* latency : 서버한테 요청을 보낼 떄 걸리는 시간
	* bandwith : 하나의 메시지에서 얼마나 많은 양을 다운받을 수 있는지

새로운 기술이 나올 수록 bandwith는 증가하고 있다. 즉, 웹사이트의 사이즈는 커질 수 있겠지. 하지만 latency는 개선이 되고 있지 않다. 물리학적인 한계가 있어서. 여기서 더 개선할 수 있는 여지는 별로 없다. 

결론적으로, http의 근본적인 문제를 해결해야 한다.
너무 많은 시간이 http 요청과 응답을 주고 받는데 낭비되고 있다는 것이다.

### Pipelining for HTTP/1.1

HTTP/1.1에서 pipelining을 도입하려는 시도도 있기는 했다.  즉, 지금 보낸 요청의 응답이 오기 전에 내가 보내야될 요청을 바로 보내는 거다.나쁘지는 않지만, 

* 구현하기 어려움
* 쉽게 깨짐
* 브라우저나 웹서버가 잘 지원되지 않음

의 이유로 지금은 거의 사용되고 있지 않다. 

그리고 만약 잘 지원된다 하더라고, 응답을 **요청을 한 순서대로 받아야하는 문제**도 있다.

이는 HOL (head of line) 문제로 유명한덱, 나중에 TCP HOL blocking issue에서 다룬다고 한다.

> HOL 문제란? [Hello IT World! :: 기타 HOL 블로킹(Head-Of-Line Blocking)](https://letitkang.tistory.com/79)


### Waterfall diagrams for web performance measurement

[image:775B9265-D042-4940-8639-FCD1D632BA70-84333-0000C4543C145F3E/CDB25875-4028-4E4E-A3B7-4F77EFB75CC1.png]

이런 폭포수 다이어그램을 잘 쓸거라고 한다. 
여기서 첫 수직선이 first point time / start render 라고 불리는, initial page를 그릴 수 있는 시간. 그리고 마지막 수직선이 끝나는 시간

그리고 크롬 개발자 도구 등의 다양한 툴들 존재

### Workarounds for HTTP/1.1 performance issues
HTTP/1.1이 근본적으로 효율적이지 못한 이유는, 요청을 보내고 응답을 기다리는데 블락이 걸리기 때문

즉, 결과적으로 **동기** 적으로 통신한다는 것이다.   만약에 네트워크나 서버가 느려지면, http의 퍼포먼스는 더욱 안좋아질 수밖에 없다.

초기에야 html document 하나 를 요청할 뿐이라 큰 이슈가 되지는 않았지만, 웹페이지가 갈수록 복잡해지고 리소스가 많아지면서 이는 문제가 되었다.

그러다보면, 자연스럽게 /web performance optimization/ 관련된 주제로 넘어가게 된다. HTTP /1.1의 단점을 극복하는게 전부는 아니지만 상당한 부분을 차지하고 있기는 하다.

HTTP /1.1 의 단점을 극복하는 방법은 크게
* 여러 HTTP connection을 사용한다.
* 적지만 잠재적으로는 더 큰 HTTP request를 만든다

요 두개로 분류된다. 다른 최적화 테크닉은 /Web Performance in Action/ 참고

### Use multiple HTTP connections
HTTP /1.1의 블로킹 이슈를 처리하는 가장 쉬운 방법은, 더 많은 커넧견을 열고, 병렬적으로 http request가 처리되도록 하는 것.

단, pipelining과는 다르게 아예 별도의 커넥션을 갖도록 해서, HOL 블로킹 이런 문제를 방지하는 것이다.  따라서, **대부분의 브라우저는 각각의 도메인에 6개 정도의 커넥션을 연다**

또한, 요 6개의 제한을 더 늘리기 위해, 많은 웹사이트들이 이미지,css, js 등의 stastic assets를 서브 도메인에서 제공한다. 그러면 해당 도메인에 대해 또 6개의 connection을 열 수 있겠지?

= /domain sharing/ 이라고 한다.  (그러다보니 브라우저는 서버가 여러개라고 생각하게 된다)

이는 단순히 병렬처리를 증가하는 것 뿐만 아니라, http headers를 줄일 수도 있다.


[image:5BE15E5B-5732-46EA-903D-6E6A75C2079B-84333-0000C6095BCE2275/4EFD8AD2-9640-462E-B2C0-E53764183C54.png]

간단하고 성능을 향상시킬 수 있어 보이지만,  단점도 존재한다.
* 클라이언트와 서버에 오버헤드가 걸린다. 
* TCP connection 여는 시간
* HTTP  커넥션 관리의 메모리, CPU 사용량 증가
* **main issue는 TCP protocol ** 과 관련된 비효율성이 근본적인 문제다. -> TCP setup and slow start
	* TCP는 모두가 알다시피 3-way-handshake 과정을 거친다. 즉, HTTP 요청 하나 보내는데 3번은 네트워크를 타야한다는 거다.
	* 뿐만 아니라, TCP의 특징.
	* CWND
	* small cwnd에서는 tcp ack을 여러번 왔다갔다 해야 full http request message를 전송할 수 있음. 
		* http response는 request보다 보통 크다보니, 이 제약에 걸림. 
		* 그다보니, 엄청 빠르고 대역폭도 높은 네트워크에서도 병목에 걸릴 수 있음. 

* 대역폭 이슈가 생길 수 있음
	* 모든 대역폭이 사용되어버리면, tcp timeout and retransmission.
* https handshaking 까지 걸림

따라서, TCP & HTTPS 까지 고려하면 비록 HTTP에서는 multi connection에서는 효율적이지 못해도 실제로 효율적이지 못함.  Multiple connection이 필요해져버림
-> HTTP /1.1의 latency를 해결하기 위한 방법이 오히려 문제를 만들어버림
-> mutlple tcp connection이 최적 효율에 이를 때면, 이미 다 로드되어서 추가 접속이 불핋요해질 수도 있음

**결론** : 여러 커넥션을 여는건.. 방법이 없을 때 아주 나쁘진 않지만 좋은 솔루션은 아니다. <- 왜 브라우저에서 6개만 허용하는지에 대한 대답이 될 수 있다.

### Make fewer requests
두 번째 optimization 방법. **요청을 더 적게 하기**
* 브라우저에 캐싱하는 방법 등을 활용해 불필요한 요청을 줄이기
* 동일한 데이터를 요청하는데 드는 http 요청을 줄이기
	* assets들을 combined file로 bundling 하기
	* 인라인 시키기

#### spriting 
작은 용량의 여러 파일들이 필요한 경우, 각각의 파일로 둬서 불필요하게 http 요청에 시간을 소모하지 말고 하나의 파일로 합쳐버린다. 그리고 css를 활용해서 각각의 이미지로 만드는 방법.
[image:70160E12-46C8-409D-AFE4-148D4785442C-84333-0000EF6190FEA8CB/BB4E2D62-B317-46DE-85B2-F1C3051A8A5E.png]


#### inlining
리소스를 다른 파일 안에 인라인 시켜버리기. Cirical css를 html 파일 안에 <style> 태그로 넣어버린다거나, svg 태그로 이미지를 바로 넣어버린다거나. 

하지만 이런 방법들의 단점들도 존재한다.
1. Complexity <- main issue : image sprite하려면 에포트가 든다..  사실 그냥 각각 서브하는게 쉽다. 
2. 파일의 낭비 : 어떤 파일에서는 정작 이미지 몇개만 쓰는데, 그걸 위해서 이미지들이 합쳐져있는 큰 파일을 다운받는다면.. 낭비 + css, js 도 새로 써줘야 함.
-> network (tcp slowstart 관점) , processing (브라우저가 쓰지도 않을 데이터를 처리해야함)  둘 다 비효율적인 방법임.
3. Caching : 캐싱된 큰 이미지 파일이 업데이트가 필요한 경우. 방문자가 더 안 쓸거여도 브라우저가 전체 파일을 다시 다운받게 해야됨. <- 뭐 버저닝을 하는 등의 방법이 있겠지만.. 쿼리스트링에 버전을 넣는다던가.. 불필요함. 낭비임

### 2.2.3 HTTP /1 performance 요약
뭐 여러가지 방법들이 있었지만, 결론적으로는 프로토콜 레벨을 바꾸는게 낫다.. 라는 내용

### 2.3 HTTP/1.1 관련 기타 이슈들
1. HTTP /1.1 은 **simple text-based protocol**임
* 물론 body에 binary를 담을 수 있지만, 결론적으로는 text based임. 헤더, 요청은 결국 텍스트니까.
* 텍스트는 사람이 알아보기는 좋다만 머신에 최적화된건 아님
* 또 보안 이슈도 있음. http attack을 보면 헤더에 새로운 라인을 추가하거나 하는 등의 방식임.

2. 텍스트 베이스이다보니, 헤더가 너무 커짐
* 사람이 읽을 수 있는 텍스트로 유지하다보니 효울적으로 인코딩하지 않음 
* 헤더가 반복됨 like cookie
* 최초에는 별 의미 없었으나, 요청이 급격하게 증가하면서 문제가됨

3. 기타 이슈들
* 성능 뿐만 아니라, 보안, 개인정보 이슈, stateless 등..

### 2.4 Real World examples
얼마나 나쁜지 함 보자!

### 2.4.1 amazon.com 예시
실제로 www.webpagetest.org에서 아마존을 쳐보면 어마어마한 waterfall 그래프가 나오는걸 볼 수 있는데.. 해석이 빡세보이니 책의 예제로 살펴보자.

[image:5122D9C2-8E56-43EC-8687-0B9CD2F03E42-84333-0000F1138F4F8894/FDDEF309-E0A6-430E-8CDD-D4772AD10C80.png]
* dns lookup time, connect, ssl/tls https negotiation 등.이 요청을 보내기 전에 필요한걸 볼 수 있다. (긴 시간은 아님)
* https 암호화 방법과 프로토콜이 개선되면 위의 ssl 시간은 줄어들 수 있겠지? 
* 무튼간에, 서버가 responsive하게 유지해서 round-trip time을 최소화하도록 보장하는 것이 중요함. 나중에 CDN 다룰 때 논의

무튼 이 초기 셋업이 끝나고 잠깐의 퍼즈를 지나고 나면 (왠지는 모르고, 크롬 이슈일 가능성 있음) http 요청이 만들어지고(연한색), html이 다운로드 되고, 파싱되고, 프로세스되어 브라우저에 나타난다.

[image:22738A1C-1900-4CD8-9E08-D2153820ABE0-84333-0000F163E9F3DB0A/45676916-BB86-414A-929D-9B2A264F6694.png]

* html 파일은 다른 css 파일들을 참조하고 있고, 그 파일들 역시 다운받아지고 있음. 
* css 파일들은 다른 도메인에서 호스팅되고 있는걸 볼 수 있음.
* 여기서, 도메인들이 지금 다르기 때문에 맨 처음에 했던 dns, connection, … 요런 과정들이 다시 이루어져야 함.  중복되는 과정.
* request 1의 요청이 끝나기도 전에 새로운 커넥션을 열어서 요청을 하고 있음. 
* 또다른 css파일을 참조하고 있기 때문에 커넥션을 또 열어야함. <- 단, dns lookup은 안필요함. request2에서 ip를 확인했으니까. 하지만,  **번거로운 tcp/ip connection과 https negotiating** 은 마찬가지로 이루어져야 함. 
* 무튼, 2번 3번 커넥션을 열어서 다운을 받고 나면, 3개의 css 파일을 더 다운받아야 함. 
이미 두개의 커넥션은 완료되어 있음. 나머지 요청들도 이제 커넥션을 열어야 함. 
* 단, 나머지 요청들 즉각적으로 바로 요청하고 있지는 않음. 아마존 소스코드를 확인해보니, 해당 소스파일들 이전에 <script> 태그가 있어서 블록킹하고 있음 <- 뭔소린지 (script 태그가 끝날 때 까지 해당 css 파일들을 다운받지 못하나봄)
<- 요게 웹의 slow performance가 http 개선만으로 전부 해결되지 않는 이유

이제는 이미지를 요청할 차례

[image:73C6EAD3-4A8F-4644-B71E-D3B5F2D3A8D9-84333-0000F28C50CF1800/8989921D-D37B-46E2-AF9B-D2DB94961E95.png]

* request 7 : 첫 번째 png 파일. Sprite file 다운받음
* request 8 ~ : 다른 jpg  파일들 다운
* 그 와중에 병렬로 다운받도록 다른 커넥션을 열어서 요청들을 날림
* request 9 10 11에서는 브라우저가 새로운 커넥션들이 더 필요할 것 같다고 생각해서 ssl 진행함. 따라서 동시에 7, 8 요청을 진행할 수 있음
* 아마존은 dns optimization을 적용해서 (dns prefetch) 

무튼.. 이렇게 로딩이 진행되는데 대충 요렇게만 봐도 많은 커넥션들이 필요하고 시간도 많이 소요되는걸 볼 수 있음.

* 아마존은 main page 에서 20개의 connection 필요, 광고 리소스를 위해 28 connectional  필요.
* 처음 6개는 잘 사용되나 나머지는 잘 사용되지도 않고 적은 리소스만 다운받는 낭비 발생

무튼, 이렇게 최적화가 꽤 잘된 곳에서도 http/1.1로  인한 성능 이슈가 있다. 또한 우회기법들이 많이 필요하다는 것도 볼 수 있다. 그리고 이런 우회기법들은 복잡하기도 하고 사용하기도 까다롭다.  그러면 아마존 정도가 되지 못하는 다른 대다수의 웹사이트는..


### 2.4.2 imgur.com
최적화가 안된 예시

### 2.4.3 
사실 다양한 우회기법들이 존재는 하나, 

* 이 우회기법들은 복잡하고, 비용이 많이 들고, 구현하기도 어렵고, 또 그 자체의 퍼포먼스 이슈들이 있음
* 안그래도 비싼 개발자들을 애초에 비효율적인 프로토콜에서 쓰는건 효율적이지가 못함. 
* 어느 순간부턴 이런 우회기법들이 안통할 수도 있음. 도메인마다 6개의 커넥션 여는 것도..

### 2.5 Moving from HTTP /1.1 to HTTP/2
HTTP-NG 가 나오기는 했으나, 1999년에 버려졌음. 너무 복잡하고 리얼 월드에 적용할 방법도 없었으니까.

### 2.5.1 SPDY
2009년 구글에서 ‘speedy’라고 부르는 새로운 프로토콜을 작업중이라고 발표.  가상이 아닌 실제 웹사이트 대상으로 실험해본 결과 64%의 성능 개선 확인.

#### SPDY
HTTPS가 HTTP를 래핑해서 사용하듯, SPDY도 마찬가지로 HTTP를 근본적으로 바꾸지는 않고 그 위에서 사용.

HTTP method, headers 그대로 존재. 로우레벨에서 작동하기 때문에, 개발자나 사람들한테는 거의 보이지 않음
-> http 요청들도 spdy로 컨버팅 딜 수 있기 때문에, 하이 레벨에서는 동일해보임. 
-> 뿐만 아니라, https 이상으로(only over) 개발되었기 때문에, 인터넷에서 패싱될때 메시지를 숨길 수 있음. 
-> 따라서, 모든 네트워크, 라우터, 스위치, 인프라 등등이 변경이 필요 없이, 심지어 spdy를 처리하고 있다는 것도 모르게 http/1 message를 처리하듯 처리 가능했음. 


HTTP-NG가 HTTP/1의 여러가지 이슈를 같이 해결하려고 했던 반면, SPDY는 딱 퍼포먼스에 초점을 맞췄음.  다음의 개념들을 도임
* Multiplexed streams : request / response가 동일한 tcp connection을 사용하되 각각의 분리된 스트림으로 패킷 그룹으로 분리됨
* Request prioritization : 동시에 모든 요청들을 보냈을 때의 퍼포먼스 이슈를 방지하기 위헤, 우선순위 개념 도입
* HTTP header compression : body는 원래 압축되고 있었고, 이제 헤더도 압축함

단, 이러한 피쳐들은 text-based에서는 불가능하기 때문에, **binary protocol**로 개발됨.

-> 큰 http message를 쪼개서 작은 message들을 하나의 커넥션에서 사용하도록
-> tcp에서 사용하는 방식을 http level에서 구현한 것.

추가 기능들도 존재
* server push :  서버가 extra resource를 담아서 응답할 수 있음. 홈페이지 요청받으면 server push가 css 파일도 제공

구글은 크롬도 자체 보유하고 있고 지네 홈페이지도 유명하다보니 이걸로 리얼월드 테스트를 할 수 있었고,  2011년 모든 구글 서비스들은 spdy가 가능해짐. 다른곳에서도 지원을 했었으나 HTTP/2.0 등장하고 망함 (?)


### 2.5.2 HTTP/2
SPDY를 통해 HTTP /1.1이  이론이 아니라 실제로 개선될 수 있다는 걸 보여줬었다.

뭐 무튼, 2012년에 IETF에서 다음 HTTP에 대한 제안을 요청을 해서, SPDY가 HTTP2의 기저를 이루었다더라.. 하는 이야기

기술적인 얘기는 4,5,7,8자에서.

### 2.6 What HTTP /2 means for web performance
그래서 뭐가 얼마나 좋아졌는데?

### 2.6.1 Extreme example of the power of HTTP/2
HTTP/2 가 83%나 빨라졌다는 얘기.

[image:D5961FE1-2C36-451C-B2F9-5C5C4C2480CE-84333-0000F5E6235F7A7E/01A4E0EC-6723-47DA-AE11-B7AFAB63DAC6.png]

Image 요청이 동시에 진행되면서 빠르게 처리가 된다.

* http2에서 이미지 다운로드 시간이 조금 더 걸린 것 처럼 보일 수도 있는데, 이는 waterfall의 측정 방식으로 인한 문제로 그렇게 보이는 것. <- queueing time은 고려가 안됨. 
> Taking request 16 as an example, under HTTP/1, this resource is requested at about the 1.2-second mark and is received 118 ms later, at approximately 1.318 ms. The browser, however, knew it needed the image after it processed the HTML and made the first request at 0.75 second, which is exactly when the HTTP/2 example requested it (not by coinci- dence!). Therefore, the 0.45-second delay isn’t accurately reflected in HTTP/1 water- fall diagrams, and arguably, the clock should start at the 0.75-second mark. As noted in section 2.4.2, Chrome’s waterfall diagram includes waiting time; so, it shows the true overall download time, which is longer than HTTP/2.

* 여러 커넥션을 사용해야 하는 HTTP /1은 queueing mechanism 문제를 발생 시킴. HTTP/2는 single connection을 사용하기 때문에, 이러한 제약을 없앨 수 있음. 
* HTTP/2의 waterfall diagram을 볼 때는, overall time을 봐야 한다는 것.

### 2.6.2 setting expectations of HTTP/2 performance gains
HTTP/2를 서또 성능 개선이 적은 겨우가 있음. 그런 경우는

1. 위에서 언급된 우회기법들에 아주 튜닝되어 있는 경우
2. 사실 HTTP /1과 관련된 성능 이슈가 아니라 다른 이슈가 있는 경우
	* 가령 너무 많은  JS, 이미지 파일들이 있는 경우 (<- process에 걸리는 시간은 HTTP/2로 개선되지 않음)

사실  사이트에서 원인을 잘 파악해야 하긴 하지만, 저자는 대체적으로 HTTP/2로 넘어가는 경우 성능은 개선될 수 있다고 함.


이전의 아마존의 예시에서 HTTP/2를 적용했을 때의 개선 사항

[image:93822E74-BBAB-4B43-8892-46A2E4B28FB3-84333-0000F6A77EF9E1C1/F36263A0-E8ED-4740-84F4-F991B3F2BCCA.png]

* load time : 보통 모든 css, js가 로드되어 unload event를 보내는 시점
* first byte :  웹사이트로부터 첫 응답을 얻는데 걸리는 시간. 보통은 리다이렉트가 아닌 첫 번쨰 진짜 응답 의미
* start render : 페이지를 그리기 시작하는 시점. 요게 가장 핵심적인 부분임. (사용자 입장에서 사이트가 안뜨면 안기다리니까)
* visually complete : 페이지가 변화하기를 멈추는 시간. js에서 synch 콜을 하고 있다면 로드 타임보다 길어짐.
* speed index : 웹 페이지의 각각의 요소가 로딩되는 평균 시간. WebPageTest에서 사용.

뭐 좋아졌다.. 라는점 확인 가능. First byt	e도 여러번 하니까 좋게 나오는 경우 확인 가능하다고 함. 오차범위 내라고 생각하면 될듯.

무튼간에, **no additional connections, less stepped waterfall** 확인 가능

### 2.6.3 Performance workarounds for HTTP/1.1 as potential anti patterns

지금까지 다룬 우회 기법들은 이론적으로는 HTTP/2에서는 사용되지 않는게 좋을 수 있음. 가령, TCP connection 1개만 사용해서 얻는 이점들은 domain shadding을 한다면 무시되겠지?
-> chapter6, connection coalescing에서 다룸. 위의 이슈를 다루는 내용

하지만, The reality is never that simple. 아직 이런 기술들을 배제하기에는 조금 이름.
* 사용자가 HTTP /1.1을 쓸 수도 있고
* 옜날 브라우저를 쓸 수 있고
* HTTP/2를 지원하지 않는 프록시를 통해 접근할 수도 있으니까.

물론, 이러한 기법들이 적용되지 않는게 좋겠지만.. 여전히 발전중이니. 

### 결론
* HTTP/1.1은 성능상, 특히 다수의 리소스를 가져오는데 근본적인 문제가 있다.
* 우회기법들이 존재하나, 그만의 단점들이 있다.
* waterfall diagram으로 특정하기 좋다.
* SPDY가 성능 이슈를 해결하기 위해 나왔다.
* HTTP/2가 SPDY의 표준화된 버전이다.
* 모든 성능 문제가 HTTP/2에서 해결되는 것은 아니다.

































