
# HTTP IN ACTION
## Part2. Using HTTP/2
1장에서 HTTP/2의 필요성, 적용방법 등을 알아봤으면, 이제부터 코어 로직을 알아봅니다.

## 4. HTTP/2 protocol basics
HTTP/2의 기본인 frame, stream, multiplexing에 대해 알아봅니다.

### Why HTTP/2 instead of HTTP/1.2?
* Binary protocol
* Multiplexed rather than synchronous
* Flow control
* Stream prioritization
* Header compression
* Server push

물론, 상위 어플리케이션 레벨에서는 2.0 이든 1.0이든 큰 차이가 체감되지는 않는다. 다만, 

이전에 다루었던 이 변화들은 근본적인 변화이기 때문에, 기존의 HTTP/1.1과 하위 호환성이 있지는 않다.  HTTP/1.0이 HTTP/1.1의 추가된 기능을 몰라도 호환은 되던것과는 다르기 때문에. 그러다보니 2.0으로 불리게 되었다.

코어 로직을 이해해서, optimization을 잘 해보도록 하자!

#### Binary rather than textual
이전에 언급된 것 처럼, HTTP/1은 text-based로 사람의 가독성은 좋지만 머신에서는 파싱의 어려움이 존재했다. 

뿐만 아니라, text-based 프로토콜에서는 요청을 보내고, 다른 요청이 처리되기 전에 응답의 전부를 받아야 했었다. 물론 개선도 있었다.

HTTP/1.1에서
* body에 binary data 추가
* chunked -encoding
* pipelining
 
등의 개선이 있기는 했지만, HOL 문제가 여전히 존재했고 파이프라이닝은 쓰이지도 못했다.

HTTP/2에서는 full-binary 프로토콜로 변화했다. 여기서, HTTP 메세지는 분명하게 정의된 (clearly-defined) **frame**의 단위로 나뉘어질 수 있다.  즉, chunked-encoding 을 아주 효율적으로 사용할 수 있으며, 표준이기도 하다.  

이 frame들은 tcp의 패킷과 유사하다.
* 모든 프레임들을 받아야 전체 HTTP message 재구성
* 하지만, TCP를 대체하지는 않고 TCP 기반으로 구성됨
* 따라서, TCP에서 보장하는 안정성 (메시지 도착, 순서 등)을 HTTP/2에서 고려하지는 않아도 됨

또한, HTTP/2의 메세지 자체는 결국 HTTP 메세지와 동일하기 때문에, 상위 어플리케이션에서는 동일하게 취급할 수 있음.

하지만 코어를 알아야 하는 이유는… 이제 적용중인 기술이다 보니 구현 이슈들을 디버깅 해야 할 수도! -I hope rare!-

#### Multiplexed rather than synchronous
HTTP/1에서는 single request-response 구조이다 보니, 블로킹 이슈가 존재했다.
따라서,

* opening multiple connection
* 더 적고, 큰 요청 보내기
와 같은 우회기법이 존재했으나,  오히려 이로 인한 문제들이 발생하곤 했었다.

HTTP/2에서는 하나의 커넥션에서 동시에 여러 요청들이 처리될 수 있게 했는데 여기서 각각의 http request-response는 서로 *다른 stream을 사용한다.*

이는, binary framing layer 덕분인데 각각의 프레임이 stream 구분자를 갖고 있다.  모든 프레임들이 도착하면 full message를 만든다.

각각의 프레임은 자기들이 각자 자기들이 어느 메시지(스트림)에 속하는지 구분하는 구분자를 가지고 있어서, 수백개의 메세지가 동시에, 그리고 Multiplexed connection에서 오고갈 수 있다. 

[image:8B5D1F12-B18C-402E-B02F-49ED13527B37-4637-00014A9BF3B13B8F/D194F4E8-4524-4EF2-9ADF-6C1850120016.png]

[image:3F9FFA8B-31FF-4A99-848D-610ADED23353-4637-00014A9F779864D2/2D0534A3-6443-4FD4-ADB9-EC9A1C52D1D8.png]

* HTTP/2에서는 HTTP/1이 커네션을 여러개 연 것에 비해, 하나의 커넥션에서 요청을 하나씩 보낸다. (HTTP/1.1의 파이프라이닝과 비슷)
* 그리고 요청을 섞어서 받을 수 있다. (파이프라이닝에서는 불가능. 순서대로 받아야함)
* 요청들이 정확하게 동시에 나가는건 아니고 순차적으로 나가기는 하는데, 이는 HTTP/1.1에서도 그렇기는 하지만, HTTP/2에서는 하나의 커넥션만 생기고 네트워크 레벨에서는 큐잉 될 것이다. (HOL 문제가 없다는 뜻?)
* 무튼간에. 요청을 보내고 응답을 받을 때 까지 blocking이 없다는게 포인트.
* 서버에서도 요청을 지맘대로 섞어서 보낼 수 있음. 물론 우선순위 부여도 가능

*Stream*

각각의 요청은 증가하는 stream ID를 부여받는데, 응답에서도 똑같은 stream ID로 돌려줌. 즉,  stream은 bidirectional! -> 요청이 끝나면 닫힘
(HTTP/1.1의 커넥션과 같지 않음. 1.1에서는 keep alive하기도 하니까)

* client에서 발생한 stream은 홀수, server에서 발생한 stream은 짝수를 받음
* server에서 stream이 발생하는건 지금은 기술적으로는 불가능하지만. 5장에서 다룸
* 요청에 대한 응답은 같은 stream ID로 마크.
* 0번은 컨트롤 스트림 (커넥션을 관리하기 위한)

무튼.. 중요한 점은

1. stream이 HTTP/1.1에서의 커넥션과 유사하다는 점. 단, 재사용되지 않고 스트림끼리 서로 독립적이지는 않다는 점.
2. 로우레벨에서 구현되어 상위에서는 알 필요가 없다는 점. 

#### Stream prioritization and flow control
기존 HTTP/1에서는 딱히 우선순위 제어가 필요 없었음 (프로토콜 자체적으로)
* request-response 구조이기 때문
* 주로 웹브라우저가 6개의 커넥션에서 우선순위를 결정 (HTTP랑 무관) , critical source 위주로

HTTP/2에서는 in-flight의 request 제한이 훨씬 늘어났기 때문에, 브라우저에 큐잉될 필요 없이 바로 보낼 수 있음
* 보통 100개
* 이런 경우, non-critical한 리소스에 대해 대역폭 낭비가 발생할 수 있음 (이미지)
** 따라서, Stream Prioritization이 필요하게됨*

*Stream Prioritization*
* 주로 서버에서 높은 우선순위의 요청에 대해 더 많은 프레임을 보내는 방식으로 구현
* 각각의 독립적인 커넥션들을 사용하는 HTTP/1 보다 더 효율적인 제어 가능
* Flow Control 가능
	* 만약 receiver의 처리속도가 sender가 메세지를 보내는 속도를 따라가지 못할 때, 패킷이 드랍되거나 재전송되어야 하는데 TCP 레벨 (connection 레벨)에서 이를 처리하지만, HTTP/2에서는 stream 레벨에서 처리 가능
	* 가령, 비디오 웹페이지의 경우, 비디오가 퍼즈되었을 때,  이 스트림만 중지하고 다른 리소스들은 다른 straem들을 통해 full capacity로 받기 가능
	* 7장에서 다룰 예정

#### Header compression
*헤더에는 중복이 많다!*
* Cookie, User-Agent(웹 브라우저), Host, Accept, Accept-Encoding
* 애네들은 요청들에서 거의 변하지가 않음. 쓸모도 없음. 
* HTTP/1에서 바디를 압축하듯 HTTP/2에서는 security issue들을 방지하는 방식으로 header를 압축함 (8장)

#### Server push
* 5장에서 다룰 예정
* 서버가 클라이언트의 요청에 하나 이상의 응답을 해줌
* 단, 필요없는걸 보내면 대역폭 낭비가 심할 수 있으니 주의가 필요함

### How an HTTP/2 connection is established
HTTP/2는 커넥션 레벨부터 기존과 다르기 때문에, client,server가 서로 HTTP/2를 쓸 수 있고 쓸거라고 합의보는 프로세스가 필요함

HTTP에서 HTTPS로 가는 경우를 HTTP/2에도 적용한다면?

1. 새로운 URL 스키마 쓰기
* http -> https:// 
* 분리에 좋았음
* HTTP/2에서도 스키마를 분리하자니,  범용적으로 HTTP/2가 도입될 때 까지는 지금 스키마를 유지해야함.  새로운 스키마를 더하면 리다이렉트가 일어날 거고, 퍼포먼스 문제가 나타날 거임. 그거 때문에 HTTP/2를 쓰는건데.
* 또 새 스키마에 맞게 링크도 바꿔야 함. 

2. 디폴트 포트 바꾸기
* 80 -> 443(https)
* 네트워크 인프라 문제 발생 가능성이 큼.  방화벽 문제 등.

따라서, *HTTP/2는 새로운 스키마를 쓰지 않음*
단, HTTP/2 connection을 맺기 위한 새로운 방법을 제시함 
* HTTPS negotiation 사용
* HTTP /Upgrade/ header 사용
* prior knowledge 사용 <- 띠용

#### Using HTTPS negotiation
HTTP/2를 지원하는건 HTTPS handshake 의 일부분으로 진행.
Redirection upgrade 이런거 안할 수 있게.

##### HTTPS handshake
우리가 아는 그 https handshake 과정

##### Application-Layer Protocol Negotiation 
[image:B4404A37-9EC1-4FDC-8EFA-8EC1DD7DFA47-4637-0001522FCE0E0CD3/E1EC5CB7-121A-437D-B275-15ED4AC6ABD8.png]


* Https negotiation 과정에서, ClientHello / ServerHello 메세지에 ALPN 옵션을 붙이기만 하면 됨
* 해당 메세지에서 클라이언트는 본인이 지원하는 프로토콜을 서버한테 알려주고, 서버가 그 중에서 하나 쓰자고 합의를 보게 됨
* 별도의 프로세스 필요 없이 HTTPS negotiation 과정에서 HTTP/2를 쓰도록 합의 가능
* 유일한 문제점은, 오래된 서버에서 지원이 안된다는 점.  (옛날 버전의 TLS 라이브러리)

##### NEXT PROTOCOL NEGOTIATION
ALPN의 predecessor. ALPN이 여기에 기반함. NPN이 많은 브라우저와 서버한테 사용되는 반면 딱히 표준화된적이 없음. (ALPN은 표준화됨)

가장 큰 차이점은 *클라이언트가 프로토콜을 정한다는 점* (ALPN은 서버가 정함)

지금 안쓰임 ㅎ

#### Example of an HTTPS Handshake with ALPN
[image:2B6C1D59-E387-49CA-B5A0-5961C05C6CDB-4637-000152AE7FC44F94/CD25E7ED-ED25-4F9E-9F1F-787510C8FDFB.png]

* 클라이언트가 h2(HTTP/2), HTTP/1.1 쓸거라고 말함
* handshake 과정을 거쳐서, 커넥션이 연결됨.  
[image:9267B2DD-C15A-4D79-9A86-76227E2C418E-4637-000152CF154AE7EC/A745DB72-FEAF-42C9-81FE-C94698749B5F.png]
* server certificater가 뜨고, curl이 HTTP/2로 바뀜. 

#### Using the HTTP upgrade header
클라이언트가 *기존에 존재한 HTTP/1.1 커넥션을 HTTP/2*로 업그레이드 해달라고 헤더에다가 Upgrade 메세지를 보내는 것.

단, HTTP에서만 사용되어야 하고, HTTPS(h2) 에서 사용되어서는 안된다. HTTPS에서는 ALPN이 사용되어야 함.
<- 브라우저에서는 HTTP/2를 HTTPS기반으로 지원하기 때문.

즉, API 제공 서버 등의 경우에만 Upgrade header 방식을 고려해야 함.
* Upgrade 헤더를 보내는 것은 전적으로 클라이언트에 달림

##### Example 1.  실패하는 경우
[image:0653F4A1-7F24-44E4-A000-F35F8C3E4873-4637-00017B2DBA42AAD8/3A749230-1B35-48A6-B83B-D6F6F08CE7D7.png]
* 클라이언트가 자기는 HTTP/2 쓸 수 있으니, 업그레이드 해달라는 요청을 보냄
* base-64로 인코딩된 세팅 메시지를 보냄
* 하지만, 만약 서버가 HTTP/2를 지원하지 않는다면, 이를 무시하고 HTTP/1.1로 응답함

##### Example2. 성공하는 경우
[image:BF4AFF53-5B8A-4F01-8B02-EF155D7A8636-4637-00017B4803FC7717/DC54F911-BF80-4E46-B160-572A653DCE03.png]
* 성공하는 경우, 위와 같은 응답을 보내고 즉시 HTTP/2로 스위치.
* SETTINGS 프레임을 보내고 원래 응답을 HTTP/2 포멧으로 보냄

##### Example3. A server suggested upgrade
[image:153B5DD2-4CB2-4702-BC0D-89EBEE420038-4637-00017B6118D49C40/7173C013-9DF3-4815-854A-98C56AB7283A.png]
* 우선, 클라이언트에서 서버는 HTTP/2를 모른다고 가정하고 일반 HTTP/1.1 요청을 보낸 경우에 해당
* 서버는 HTTP/2가 지원된다는 사실을 알리고자, Upgrade 헤더를 담아 응답함
* 즉, /suggestion/ (프로토콜 전환 요청은 클라이언트에서 시작되어야 하기 때문에)
* 여기서 서버는 h2c, h2가 가능하다고 응답을 보냈는데, 만약 클라이언트에서 h2를 사용하고 싶다면 반드시 HTTPS로 전환후 ALPN negotiation을 거쳐야 함.  Upgrade header는 h2c에서만 사용 가능하기 떄문

#### Issues with sending an upgrade header
웹 브라우저에서는 HTTPS 기반으로만 HTTP/2를 지원하기 때문에, Upgrade header 방식은 사실 웹 브라우저가 거의 사용하지는 않음. 왜?

1. 웹서버가 HTTP/2를 지원하지 않을 수 있음
* 브라우저 - 웹서버 - WAS 의 구조 가정
* 웹서버가 사실상 리버스 프록시의 역할을 함
* 만약 여기서 WAS가 브라우저한테 “HTTP/2 쓸래?” 하고 suggestion을 보내고, 웹서버가 이를 무지성으로 포워딩 한 다음, 클라이언트가 ok 하고 HTTP/2를 쓰기로 함
* 하지만, 만약 웹서버가 구형인 경우, 정작 클라이언트와 연결된 웹서버는 HTTP/2를 지원하지 않을 수 있음

2. 클라이언트와 WAS의 입장이 다를 수 있음
* 웹서버가 클라이언트와는 HTTP/2로 통신을 하고, WAS에게는 HTTP/1.1로 프록싱 하고 있는 경우
* 여기서 WAS가 suggest를 해버리게 되면, 클라이언트 입장에서는 이미 HTTP/2를 쓰고 있는데 쟨 뭐라는거야 하면서 혼란이 생길 수 있음

nginx에서는 리버스 프록시 등으로 WAS 앞에 붙을 때 Upgrade header를 없애기도 함.  (proxy_hide_header Upgrade)

저자는 suggestion을 보내지 않는 application을 선호하며, 아파치 같은 경우는 
Header unset Upgrade 를 사용해 이 기능을 꺼버릴 수 있음. 
(HTTP/2를 지원하는 아파치를 쓸 때는 사용하는 것이 좋음)


### Using prior knowledge
클라이언트가 만약 서버가 HTTP/2를 지원한다는걸 알고 있으면, 별도의 절차 없이 바로 HTTP/2를 사용하는 것.

	* ALt-Svc Header, ALTSVC 프레임 등에서 advertised 되거나..
	* 가장 risky한 방법. 거절 메세지를 잘 처리해야 함.  사전 지식이 틀릴 수도 있으니
	* 클라이언트, 서버를 다 통제하고 있을 때 사용되어야 함

### HTTP Alternative Services
HTTP/2 스펙에 정의되지 않은, 약간 숨겨진 방법은 HTTP Alternative Services를 사용하는 것. (HTTP/2가 릴리즈 된 이후 별도의 스탠다드로 나옴)

여기서는 HTTP/1.1을 쓰고 있는 클라이언트에게, 서버가 요청받은 리소스가 다른 location에서 다른 프로토콜로 지원된다는걸 알릴 수 있도록 함.  Prior knowledge와 함게 사용 가능

* HTTP/2 커넥션이 이미 있는 경우에도 사용이 가능함. 만약 클라이언트가 다른 커넥션으로 스위칭 하고 싶은 경우
* 널리 쓰이지는 않고, 하나의 커넥션에서 다른 커넥션으로 스위칭 해야하기 때문에 HTTP/2 ALPN이나 prior knowledge를 쓰는 것보다 느림. 

### The HTTP/2 preface message
HTTP/2 커넥션에서 가장 먼저 보내져야 하는 메세지는 	HTTP/2 connection preface

[image:F0F81017-CAB4-40DC-B2A2-CCEB22347F76-4637-00017D3880C84A19/4F813E06-AF4D-43C5-853F-ED3619CE7356.png]

* 클라이언트가 보내는 메세지인데, hex annotation

[image:CF2FC050-6EFB-43B2-8E64-D202558038C4-4637-00017D44B3BE3E9D/72209388-F3CD-4ACD-AF9C-0A85BEA51078.png]

* 아스키로는 이런 모양이고, 이는 사실 HTTP/1 스타일의 메세지임

[image:C32FF0B1-DC1C-4588-8F6D-ECF7D5DD3218-4637-00017D4B21ADA0B6/02F1E595-8E4F-4DCC-A03E-6AA64A9D6F9E.png]

* PRI : Http methid
* * : resource
* HTTP version
* SM : request body
ㄴ 원래는 FOO, BAR,BA 요런게 쓰였다고 함. 딱히 왜 쓰였는지는 공식 스펙에 안쓰여 있다고 함. 

사실 이런 nonsensical한 메세지는 HTTP/2를 못알아듣는 서버한테 클라이언트가 HTTP/2를 요청하는 경우를 위한 것.

그런 서버는 이걸 그냥 파싱하고 못알아들어서 메세지를 거절할 거임.

만약 이걸 알아듣는 서버는 SETTING frame을 보낸다.

### HTTP/2 frames
HTTP/2 커넥션에서는, 하나의 multiplexed 커넥션에서 여러 스트림에 frame들을 보내는 방식으로 이루어짐.

웹개발자가 알 필요는 없기는 한데 한번 잘 알아보자..

#### Viewing HTTP/2 frames
* Chrome net-export 
-> chrome://net-export/
-> Start Logging to Disk
-> HTTPS 페이지 염
-> Stop Logging
-> https://netlog-viewer.appspot.com
-> 로그파일 열기
[image:D7CAC6C2-F9E0-41E7-9A24-215F336DE4E6-4637-00017E052BE47F93/1BA2AA16-ECBA-4E18-8F78-6E9F1CB9F385.png]

이런 식..
[image:C5F37C8B-CCE8-4542-BE0D-460BAE22B807-4637-00017E1328C5F3CD/스크린샷 2022-04-12 오후 8.04.37.png]
SETTINGS 프레임의 부분


* wireshark 등등의 많은 툴들..

#### HTTP/2 frame format
HTTP/2 프레임은 고정 길이의 헤더로 이루어짐
[image:EBAA420A-5EB3-40B6-BE60-08A268EFC796-4637-00017F5A64E762AC/7E5DD647-B053-4D5F-B118-4E422F82475E.png]
* HTTP/1이 고정되지 않은 길이의 텍스트 메세지로 이루어진 반면 , HTTP/2 프레임은 엄격하고 잘 정의된 포멧이 있음. 
* 따라서, 기존에는 줄바꿈, 띄어쓰기 등등 파싱하면서 비효율적이고 에러가 생길수도 있던 반면,
* 특정 코드를 쓰거나 하는 식으로 더 작아지고 파싱하기도 쉬워짐
> Byte가 아키텍쳐에 따라 모호한 면이 있기 때문에 본문에서는 8bit -> Octet 사용


### Examining HTTP/2 message flow by example
nghttp로 요청을 보내는 예제. HTTPS setup이랑 HTTP/2 preface 를 보여주지 않기 때문에, 바로 SETTINGS 프레임을 먼저 받음

[image:25C1CA4E-F185-4928-8C81-E7ADD463CE3C-4637-00017FBDF9811874/AB52E1D9-6614-48CB-8A5D-1C29B1EDBEFE.png]


#### SETTINGS FRAME
* SETTING 프레임은 서버와 클라이언트가 HTTP/2 preface 이후 반드시 첫 번째로 보내야 함. 
* 빈 payload와 몇몇 field/value pair로 구성
* SETTING FRAME은 flag중에서 ACK만 셋팅해줌.
	* 만약 이쪽 (this side : client, server 인듯?) 에서 Setting을 전파하면 0로 만들고, 만약 반대쪽에서 셋팅을 했다면 거기에 대한 ACK으로 1로 만들어줌.

[image:8C2B404F-C6D2-4918-8F95-2ED71A95D3AB-4637-00017FC8D8D5A42B/D7ABC7EB-6C70-4956-9B8C-3678BD0AF58F.png]

이걸 해석해보면, 받은 SETTING 프레임은 
* 30 octet  길이
* 플래그가 없음 (따라서, acknowledgement fame이 아님)
* steram id 는 0 <- 예약 id. Control message(SETTING, WINDOW_UPDATE)에 쓰임.

그리고 이제 플래그를 해석해보자면, (순서는 섞여도 됨)

1. SETTING_HEADER_TABLE_SIZE 
* HPACK HTTP header compression에 사용, chapter8
2. SETTINGS_MAX_FRAME_SIZE
* client가 이 이상 큰 페이로드를 보내면 안됨
3. SETTINGS_MAX_HEADER_LIST_SIZE
* 저거보다 더 크고 압축이 안된 헤더를 보낼 수 없음
4. SETTINGS_MAX_CONCURRENT_STREAMS
* 이전에 100개 스트림 제한이 있는 서버에서 100개 이상의 이미지를 로드하는 경우 예시
* 이 때, 요청은 큐에 쌓이면서 free stream을 기다리나 HTTP/1에 비해 더 적은 커넥션 제한으로 함. 
* HTTP/2에서는 병렬로 요청이 처리되는 수를 극적으로 늘렸으나, 서버에 의해 100, 128 stream으로 제한 되는 경우가 많음
5. SETTINGS_INITIAL_WINDOW_SIZE
* flow control

사실 default initial value를 가지고 있는 경우가 많기 때문에, 축약된 버전을 보내기도 함.

SETTINGS_ENABLE_PUSH <- 요런거는 클라이언트에서 쓰는거지, 서버에서 쓰는건 아님. 서버 푸시 허용 여부라.

[image:1AF6E24E-EA72-47C7-B71C-40F3A548B05A-4637-000180CC5E83362E/F4D6806E-E04A-4126-8CAF-269F01FC0E9A.png]

그 뒤에는 ack 를 주고받는 모습. flag를 1로 세팅

#### WINDOW_UPDATE frame
[image:339C3B6D-6485-4E8D-8157-4ACB5D594FE4-4637-000180D7912B0AA0/13AB0A4C-66C7-4960-8F8F-2937AEE8DB50.png]
Flow control에 사용
* 보낼 수 있는 데이터의 양을 제한
* HTTP/1에서는 TCP flow control
* HTTP/2에서는 하나의 커넥션에 여러개의 stream이 있기 때문에, tcp 플로우만으로는 해결이 안됨. 따라서, 각각의 stream에 대한 slow-down 메서드가 필요함
* window size의 초기 값은 SETTINGS FRAME에서 정하고, WINDOW_UPDATE 프레임에서는 이 양을 증가시킴. 그래서 간단함
* 만약 stream 0에 적용한다면, 이건 HTTP/2 connection 전체에 적용됨
* 그리고 이 flow control은 *DATA frmae* 에만 적용됨.  <- 다른 중요한 컨트롤 메세지가 데이터 프레임에 블락킹될 수도 있으니. 즉, 데이터 프레임만 가변적임.

#### Priority Frame
[image:597CD24E-0EA2-460F-8C20-677BBC9266AB-4637-0001812A256A6AB5/0ABA8C39-E8FE-4254-AAF8-75DE3FF77936.png]
* 클라이언트인 nghttp가 사용할 다양한 priority를 가진 스트림을 생성함
* nghttp는 3-11 스트림을 직접 사용하지 않고, dep_stram_id로 걸어놓음(?)
* 이렇게 하면 미리 만들어진 나중에 새로 만들어진 스트림에 각각 우선순위를 직접 명시하지 않으면서도 사용할 수 있다 (?) 뭔소리지
* 파이어폭스 모델을 써서 있는거고 모든 클라이언트들이 이렇게 스트림을 미리 만들지는 않는다함
* 지금은 그냥 넘어가고, 중요한 리소스들에 대해 우선순위 제어가 있을 수 있다는 점만 알고 넘어가자

#### HEADERS FRAME
지금까지 셋업이었고… 이제부터 HTTP/2 request.

HTTP/2 request는 HEADER frame에 담김

[image:D37E52CE-795D-4E28-BFA0-CC8134E9813D-4637-0001818D32C997E4/D9BC9FBA-6AFB-4D3E-BACE-2E3856AE35AF.png]

사실 위의 몇줄 제외하면 HTTP/1에서 보던 포멧이긴 한데, HTTP/1에서 첫 줄과 host 를 넣었음. 반면,

* HTTP/2에서는 전부다 헤더에 있고, 새로 pseudoheader가 도입됨. 위에서 : 으로 시작되느 부분들이 해당.
:authority -> HTTP/1에서의 HOST를 대체. 

*단, pseudoheader들은 내가 맘대로 추가하거나 넣을 수가 없음!* 
* HTTP/2를 바꾸지 않는 이상.  엄격하게 정해져 있음. 
* * 만약, 필요하다면, HTTP/2스펙이 바뀌어야 하고, 그러면 새로운 SETTING 파라미터가 필요함.
* lower case만 사용해야 함. 
* 뭐 각종 bad formatting을 막기 위해 엄격한 규칙 적용.

[image:AB7E8E12-5404-4719-BC56-76A2EFFD7981-4637-00018273815E50F3/6E247950-3CBE-47A8-94D2-BA05C18CEEAB.png]

* E, Stream Dependency, Weight : 나중에
* Pad Length, Padding : 보안의 이유로 메세지의 길이를 숨기도록
	* Block Header Fragment : pseudoheader를 포함한 헤더들이 담기는 곳. 지금은 tool에 의해 decompress 되기는 하는데, 8장에서 다룸

HEADER frame은 4개의 플래그를 설정함

1. END_STREAM
* 지금 HEADER frame 뒤에 아무것도 없을 때 셋팅됨. 
* CONTINUATION frame은 예외. 

2. END_HEADERS
* 모든 HTTP header가 지금 이 프레임에 다 담겨있고, CONTINUATION 프레임에 이어서 들어오지 않는다는 의미

3. PADDED
* padding이 사용될 때 셋됨. 

4. Priority
* E, Stream Dependency, Weight 필드가 지금 프레임에서 세팅되었다는 의미

만약 HTTP header가 single frame보다 큰 경우에는?
-> CONTINUATION frame에 담김. HEADERS 직후에 들어옴
* 복잡할 수도 있는데, 다른 필드들은 한번만 셋팅되어야 하기 때문에 HEADER가 계속 들어오면 문제가 될 수 있음. 
* 사실 이런 일은 거의 없기는 함

#### DATA frame
바디에 해당하는 부분. HTTP/1에서는 헤더 아래에 두칸 띄고 담겼지만, HTTP/2에서는 별도의 메시지 타입으로 분류됨. 
-> Response를 여러 부분으로 쪼개면서 하나의 커넥션에서 multiplexed stream을 가능하게 함.

구조는 간단함.  길이도 가변적임. 어떤 데이터든 담길 수 있음

[image:9BCFAC1A-D607-47B0-BEAF-63F0AD7018DE-4637-0001833690FF5EAC/72205F02-6D90-4ED8-8C10-E80A1C79BCBC.png]

또한, 두 개의 플래그를 지정함

1. END_STREAM : 지금 프레임이 스트림에서 마지막인 경우 세팅해줌
2. PADDRE : DATA Frame의 첫 8 비트가 프레임 끝에 얼마나 많은 패딩이 더해졌는지 알려주는 용도

[image:44D9DDB1-DE8A-4775-AF80-8ABAAD2AB2B3-4637-000183542822442D/2819A12C-F8EC-4552-BD44-7E1988DE981C.png]
* 데이터 프레임에 담긴 점
* 클라이언트가 처리하면서 WINDOW_UPDATE 프레임을 보내는 점. 
* 페이스북에서는 특이하게 클라이언트가 많은 양을 처리할 수 있는데 적게 보낸다는 점. 
* 프레임이 자연스럽게 여러 부분으로 나뉘기 때문에, chunked encoding이 딱히 필요가 없다는 점. 

#### GOAWAY Frame
더 보낼 메세지가 없거나, 심각한 에러가 발생해서 연결을 끊을 때 사용

[image:BB920D58-9A1A-4129-8CFE-DA84806CAA6B-4637-0001838DD5F78971/2AFA23B3-3488-4105-99D4-6219DA1EF241.png]

딱히 플래그를 변경하지는 않음. 

현재 예제에서는, nghttp가 홈페이지 HTML을 요청을 했을 때, 서버에서는 보통 브라우저들이 요청하는 CSS, JS 등등을 같이 보내주는데, 지금 nghttp에서는 필요가 없어서 커넥션을 끝내는 경우. 

### Other frames
1. CONTINUATION frame
* 위에서 언급된 것 처럼 큰 HTTP header에 사용, HEADERS 프레임 or PUSH_PROMISE 뒤에 바로 따라옴.
* 헤더가 다 있어야 요청이 처리되고, HPACK dictionary 를 유지하기 위해서 바로 따라와야됨. 
* 이게 멀티플렉싱의 요소를 저해하다 보니 필요하다 안필요하다 말이 많았는데 (혹은 헤더 프레임이 더 커져야 하거나) 지금은 잘 안쓰임

2. PING FRAME
* round-trip 시간을 재거나 혹은 안 쓰는 커넥션을 유지할 때 사용
* 만약 이 프레임을 받으면, 즉시 동일한 PING 프레임으로 응답해야 함. Control stream (0번) 에서 사용됨. 

3. PUSH_PROMIS FRAME
* 클라이언트가 직접 요청한 리소스 외에, 서버에서 server push 할거라고 말할 때 사용됨.
* 푸시할 리소스에 대한 정보를 알려줘야 해서, HEADERS 프레임에 포함될 header들은 다 포함함. 

4. RST_STREAM FRAME
* 즉시 스트림을 취소하거나 리셋할 때 사용
* 에러, 혹은 요청이 더이상 쓸모 없을 때. 
	* 가령, 클라이언트가 이미 딴대로 가버리거나 (Navigated away), 로딩을 취소하거나, 서버 푸시 리소스가 필요 없을 때
* HTTP/1에서는 이런 기능이 없어서 뭐 어쨋든간에 블락되었어야 했는데, 이 기능이 추가됨

5. ALTSVC 프레임
* 첫 번째로 스펙에 추가된 프레임.
* 이전에 언급된 것과 같이, 특정 리소스를 fetch 할 수 있는 다른 서비스가 있다는 것을 전파
* upgrade에 사용

6. Origin frame
* 2018년에 추가된 프레임. 어떤 origin (domain name 등)에 응답할 것인지 서버가 나타낼 수 있게 하는 프레임.

7. CACHE_DIGEST frame
* 새로운 프레임 제안 (proposal)
* 클라이언트가 지금 지가 어떤 asset을 가지고 있는지 표현할 수 있게 해줌.
* 즉, 서버가 이 리소스들은 푸시하면 안된다라는 표시
* 


















