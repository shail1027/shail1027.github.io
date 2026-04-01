---
title: "크롬 확장 프로그램 동작 원리"
author: Lee Yebin
date: 2026-04-01 10:25:27 +0900
categories: [TECH, Frontend]
tags: [chrome_extension, JavaScript]
pin: false
math: true
mermaid: true
# image:
#   path: 
#   lqip: 
#   alt:
---

## 확장프로그램은 어떻게 웹페이지에 접근하나?

결론부터 말하면 — **브라우저가 직접 JS를 페이지에 심는 것이다.**

일반 웹페이지의 JS는 해당 도메인의 리소스에만 접근할 수 있다(Same-Origin Policy). 하지만 크롬 확장프로그램은 브라우저 레벨에서 동작하기 때문에 이 제약을 벗어날 수 있다. 대신 사용자에게 명시적으로 권한을 요청해야 한다.

---

## 권한의 종류

### 1. API 권한 (permissions)

Chrome이 제공하는 특수 API를 사용하기 위한 권한. `manifest.json`에 명시한다.

```json
{
  "permissions": [
    "storage",      // chrome.storage API 사용
    "tabs",         // 탭 정보 읽기/제어
    "activeTab",    // 현재 활성 탭에만 접근 (더 제한적, 권장)
    "scripting",    // 페이지에 JS/CSS 동적 주입
    "alarms",       // 타이머/스케줄링
    "notifications" // 시스템 알림
  ]
}
```

이걸 선언하지 않으면 해당 API 호출 시 그냥 에러가 난다. Chrome이 API 자체를 막아버린다.

### 2. 호스트 권한 (host_permissions)

어떤 도메인의 페이지에 접근할 수 있는지 선언한다.

```json
{
  "host_permissions": [
    "https://www.google.com/*",   // 구글만
    "https://*.github.com/*",     // 깃헙 전체 서브도메인
    "<all_urls>"                   // 모든 사이트 (가장 강력, 가장 위험)
  ]
}
```

호스트 권한 없이 특정 사이트의 DOM을 읽거나 fetch를 날리면 차단된다.

### 3. 선택적 권한 (optional_permissions)

설치 시점이 아니라 런타임에 사용자에게 요청하는 권한.

```js
// 버튼 클릭 같은 사용자 액션 내에서만 요청 가능
chrome.permissions.request(
  { permissions: ["tabs"], origins: ["https://example.com/*"] },
  (granted) => { console.log(granted ? "허용됨" : "거부됨"); }
);
```

처음부터 `<all_urls>` 권한을 요구하면 사용자가 설치를 꺼릴 수 있다. 필요한 순간에만 요청하는 게 UX에 좋다.



## Content Script — 페이지에 JS를 심는 방법

Content Script는 크롬이 웹페이지 로드 시 자동으로 주입하는 JS 파일이다. 마치 해당 페이지가 원래부터 그 스크립트를 가지고 있었던 것처럼 동작한다.

```json
// manifest.json
{
  "content_scripts": [
    {
      "matches": ["https://www.youtube.com/*"], // 이 URL 패턴에서만 실행
      "js": ["content.js"],                      // 주입할 JS
      "css": ["content.css"],                    // 주입할 CSS
      "run_at": "document_idle"                  // 언제 주입할지
    }
  ]
}
```

`run_at` 옵션:
- `document_start` → HTML 파싱 시작 전 (DOM 없음)
- `document_end` → DOM 준비됨, 리소스 로드 전
- `document_idle` → 페이지 완전히 로드 후 (기본값, 대부분 이걸 씀)

### Content Script가 할 수 있는 것

```js
// content.js — 유튜브 페이지에 주입된 상태

// DOM 읽기/수정
document.querySelector("#title").style.color = "red";

// 페이지의 이벤트 감지
document.addEventListener("click", (e) => console.log(e.target));

// chrome.storage 접근
chrome.storage.local.get("myData", (data) => console.log(data));

// 확장프로그램 내 다른 컨텍스트와 메시지 통신
chrome.runtime.sendMessage({ type: "PAGE_LOADED", url: location.href });

// 페이지의 JS 변수에 직접 접근 불가
console.log(window.somePageVariable); // undefined — 격리되어 있음
```

### Isolated World

Content Script는 페이지와 **같은 DOM을 공유하지만, JS 실행 환경은 완전히 분리**된다.

```
웹페이지 JS 세계                Content Script 세계
─────────────────               ──────────────────────
window.myVar = "hello"          window.myVar → undefined
document.body ──────────────── document.body (공유!)
fetch(), XHR                    fetch(), XHR (별도 실행)
```

이 격리 덕분에 페이지의 JS가 확장프로그램을 건드리거나, 확장프로그램이 페이지 JS를 오염시키는 걸 방지한다.

만약 페이지의 JS 컨텍스트에 접근해야 한다면 `<script>` 태그를 직접 DOM에 삽입하는 우회 방법이 있다.

```js
// content.js
const script = document.createElement("script");
script.src = chrome.runtime.getURL("injected.js"); // 페이지 컨텍스트에서 실행됨
document.head.appendChild(script);
```

---

## 동적 주입 — scripting API

manifest에 미리 선언하지 않고 코드로 직접 주입하는 방법도 있다. `scripting` 권한이 필요하다.

```js
// background.js 또는 popup.js
chrome.scripting.executeScript({
  target: { tabId: tab.id },
  func: () => {
    // 이 함수가 해당 탭 페이지에서 실행됨
    document.body.style.background = "hotpink";
  }
});

// CSS 주입
chrome.scripting.insertCSS({
  target: { tabId: tab.id },
  css: "body { font-family: monospace !important; }"
});
```

---

## 권한과 보안 — 왜 이게 중요한가

크롬 확장프로그램은 설치하는 순간 선언된 권한 범위 내에서 엄청난 권한을 갖는다.

```
<all_urls> + scripting 권한을 가진 확장프로그램이 할 수 있는 것:
  - 모든 사이트의 DOM 읽기/수정
  - 입력 폼의 값 (비밀번호 포함) 읽기
  - 쿠키/세션 탈취
  - 페이지 내용을 외부 서버로 전송
```

실제로 악성 확장프로그램이 이런 방식으로 계정을 탈취하는 사례가 많다. 그래서 Chrome 웹 스토어는 권한 범위가 넓은 확장프로그램을 설치할 때 경고를 보여준다.

확장프로그램 개발 시 권한 최소화 원칙:
- `<all_urls>` 대신 필요한 도메인만 명시
- `tabs` 대신 `activeTab` (현재 탭에만 접근)
- 필요할 때만 `optional_permissions`로 요청

