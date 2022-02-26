# Part 5 URL 단축기 설계

## 1. 문제 이해 및 설계 범위 확정

TIP: 시스템 설계 면접 문제는 의도적으로 어떤 정해진 결말을 갖지 않기 때문에
질문을 통해 모호함을 줄여야 한다.

e.g.

Q: 어떻게 동작하는지 예제 보여줘

A: `https://www.systeminterview.com/q=chatsystem&c=loggedin&v=v3&l=long` 이런
입력이 주어졌을 때 이 서비스는 `https://tinyurl.com/y7ke-ocwj` 같은 단축 url을
제공해야 한다.

Q: 트래픽 규모는 어느 정도?

A: 매일 1억개 (100M)의 단축 url 생성 가능해야

Q: 단축 url의 길이는 어느 정도?

A: 짧으면 짧을수록 좋다.

Q: 단축 url에 포함될 문자에 제한이 있나?

A: 0-9, a-z, A-Z

Q: 단축 url을 시스템에서 지우거나 갱신할 수 있나?

A: 노노

이 시스템의 기본적 기능은:

1. url 단축
2. url redirection
3. 높은 가용성(availability), 규모 확장성(scalability), 장애 감내(Fault Tolerant)

### 개략적 추정

- 쓰기 연산: 매일 1억개 단축 url 생성
- 초당 쓰기 연산 1억/24/3600 = 1160
- 읽기 연산: 읽기 연산과 쓰기 연산 비율이 10:1 이라고 할 때 읽기 연산은 초당
  11600 회 발생.
- url 단축 서비스를 10년간 운영한다고 가정하면 1억*365*10 = 3650억 (365B)개의
  레코드 보관해야
- 축약 전 url의 평균 길이는 100이라 가정
- 따라서 10년간 필요한 저장 용량은 3650억 * 100바이트 = 26.5 TB

계산 후 면접관과 합의.

## 2. 개략적 설계안 제시 및 동의 구하기

### API Endpoint

- RESTful API 스타일
- URL 단축용 엔트포인트

  - *POST /api/v1/data/shorten*
  - param: {longURL: longURLstring}
  - return: shortened URL

- URL 리디렉션용 엔드포인트
  - *GET /api/v1/shortUrl*
  - return: original URL

### URL Redirection

Location header에 원래 url이 담겨있다.
![8-1](/part8/images/8-1.png)

![8-2](/part8/images/8-2.png)

**301과 302 응답의 차이**

301 Permanently Moved: 해당 url에 대한 요청 처리 책임이 영구적으로 Location
헤더에 반환된 url로 이전되었다는 응답. 따라서 브라우저가 이 응답을 캐싱한다.
추후에 같은 단축 url로 요청 보낼때 캐시된 것 사용. 서버 부하를 줄여야 할 때 좋다.

302 Found: 주어진 url로의 요청이 "일시적"으로 Location 헤더가 지정하는 url에
의해 처리되어야한다는 응답. 따라서 클라이언트 요청은 항상 단축url 서버에 보내진
후 리디렉션 된다. 트래픽 분석이 중요할 때. 클릭 발생률이나 위치 추적에 유리.

URL Redirection 구현하는 가장 직관적인 방법은 해시 테이블 사용하는 것.
해시 테이블에 <단축url, 원래url> 쌍 저장.

- 원래url = hashTable.get(단축url)
- 301 or 302 response with Location header.

### URL 단축

단축 url이 www.tinyurl.com/{hashValue} 같은 형태라고 할 때, 해시 함수를 찾는
것이 중요하다.
![8-3](/part8/images/8-3.png)

해시 함수 요구사항

- 입력으로 주어지는 긴 url이 다른 값이면 해시 값도 달라야 함.
- 계산된 해시 값은 원래 입력으로 주어졌던 긴 url로 복원될 수 있어야.

## 3. 상세 설계

### 데이터 모델

사실 모든 데이터를 해시 테이블에 두는 것은 곤란하다.(메모리는 한정적이고 비쌈.)
따라서 더 나은 방법은 <단축url, 원래url> 쌍을 관계형 데이터베이스에 저장하는
것이다.

필수 칼럼:

- id
- shortURL
- longURL

### 해시 함수

해시함수가 반환하는 단축url을 hashValue라고 하겠다.

#### 해시 값 길이

hashValue 는 [0-9, a-z, A-Z] 즉, 10+26+26=62개의 문자만 사용 가능.
hashValue의 길이를 정하기 위해서는 62^n >= 3650억(365B)인 n의 최솟값을 찾아야함.

![table-8-1](/part8/images/table-8-1.png)
따라서 해시발류의 길이는 7.

#### 해시 함수 구현: 해시 후 충돌 해소 VS BASE-62 변환

##### 해시 후 충돌 해소

잘 알려진 해시 함수를 사용하여 `https://en.wikipedia.org/wiki/Systems-design` 을
축약한 결과.

![table-8-2](/part8/images/table-8-2.png)
그치만 전부 7보다 길다. 어떻게 하면 길이를 줄일 수 있을까?

계산된 해시값에서 처음 7개 글자만 이용.
-> 해시 결과 충돌 가능성 업업. 충돌이 발생했을 때 충돌 해소될때까지 사전에 정한
문자열을 해시값에 덧붙임.

  ![8-5](/part8/images/8-5.png)
  이 방법은 충돌 해소할 수 있긴 하지만 단축url 생성시 데이터베이스 쿼리를 1번
  이상 해야하므로 오버헤드가 크다. 데이터베이스 대신 블룸 필터를 사용하면 성능
  높이기 가능.
  
  블룸 필터: 어떤 집합에 특정 원소가 있는지 검사할 수 있도록 하는 확률론에
  기초한 공간 효율이 좋은 기술.
  
##### BASE-62 변환

진법 변환 (Base conversion): 수의 표현 방식이 다른 두 시스템이 같은 수를
공유하여야 하는 경우에 유용. 62 진법을 사용하는 이유는 해시발류에서 쓸 수 있는
문자 개수가 62개이기 때문이다.

0-0, ..., 9-9, 10-a, 11-b, ..., 35-z, 36-A, ..., 61-Z

10진수 11157을 62진수로 변환하는 예시
![8-6](/part8/images/8-6.png)
따라서 단축 url은 `https://tinyurl.com/2TX`가 된다.

**두 접근법 비교**
![table-8-3](/part8/images/table-8-3.png)
해시 후 충돌은 url을 바로 변환하는 방식이고 BASE-62는 id를 변환하는 것이다.

### url 단축기 상세 설계

BASE-62사용.
![8-7](/part8/images/8-7.png)

e.g.

1. 입력 url이 `https://en.wikipedia.org/wiki/Systems-design`라고 하자.

2. id 생성기가 2009215674938이라는 아이디 반환.

3. 이 id를 62진수로 변환하면 zn9edcu 반환.

4. 데이터베이스 기입.
    ![table-8-4](/part8/images/table-8-4.png)

이전 챕터의 분산환경에서의 아이디 생성기 구현 방법을 기억하자.

### url 리디렉션 상세 설계

읽기보다 쓰기를 더 자주 하는 시스템이라 캐시를 사용하여 성능 높이기.
![8-8](/part8/images/8-8.png)

## 4. 마무리

시간이 남는 경우 면접관과 이야기해 볼 것

- 처리율 제한 장치(rate limiter)
  - 지금 설계는 많은 양의 요청이 몰려올 경우 무력화 될 수 있다.
- 웹 서버 규모 확장: 이 설계의 웹 계층은 stateless기 때문에 서버를 자유롭게
  증설/삭제 가능.
- 데이터베이스 규모 확장: DB다중화나 sharding하여 규모 확장 달성
- 데이터 분석 솔루션: 어떤 링크를 얼마나 많은 사용자가 언제 주로 클릭했는지 등.
- 가용성, 데이터 일관성, 안정성: 대규모 시스템의 필수 속성들
