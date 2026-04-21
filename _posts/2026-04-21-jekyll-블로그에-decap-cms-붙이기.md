---
title: Jekyll 블로그에 Decap CMS 붙이기
date: 2026-04-21T13:18:00.000+09:00
categories:
  - TECH
  - GitHub
tags:
  - Gitio
  - Jekyll
---
## **개요**

여러 번의 블로그 플랫폼을 옮긴 후 겨우 gitio에 정착을 했는데, 최근까지 글을 꾸준히 쓰다가 `md` 파일을 직접 편집하고 쓰는 게 점점 버거워졌다. 그래서 처음에는 jekyll 템플릿을 직접 수정해서 다른 블로그 플랫폼처럼 에디터를 만들어보면 어떨까~ 해서 찾아봤는데 우선 파일이나 로그인등을 저장할 백엔드 서버랑 db가 없고, github pages는 정적 페이지만 호스팅되는 걸 알고 다른 방법을 찾아봤더나 `Decap CMS`라는 게 있어서 한 번 적용해보기로 했다.

Decap CMS는 Netlify CMS에서 이름이 바뀐 오픈소스 CMS로, GitHub 레포지토리를 백엔드로 사용한다. 설정 파일만 추가하면 `https://yoursite.com/admin/`에서 글을 작성하고 바로 커밋까지 해준다.

여담으로 우리가 흔히 생각하는 블로그나 타 글쓰기 플랫폼의 에디터를 WYSIWYG(위지윅: What You See Is What You Get)라고 부르는 듯하다. 간단히, 문서 편집 과정에서 환면에 포맷된 낱말, 문장등이 출력과 동일하게 나오는 방식이다. 따라서 WYSIWYG Editor는 편집 화면에서 입력한 글자, 그림 등의 컨텐츠 모양이 그대로 최종산물 화면상에서, 또는 출력물에서 나타나도록 하는 에디터이다. 

- - -

## 구조 이해

Decap CMS는 GitHub API를 통해 파일을 읽고 쓴다. 문제는 GitHub OAuth 인증에 서버 사이드 코드가 필요하다는 점이다. GitHub Pages는 정적 호스팅이라 서버가 없기 때문에, **OAuth 프록시**를 별도로 띄워야 한다.
예전엔 `decap-proxy.netlify.app`이라는 공용 프록시가 있었지만 지금은 없어졌고, 현재 가장 간단한 방법은 **Cloudflare Workers**로 직접 배포하는 것이다. 무료 플랜으로 충분하고 24시간 돌아간다.
ㅊ

- - -

### 1단계: GitHub OAuth App 등록

GitHub -> Settings -> Developer settings -> OAuth Apps -> New OAuth App에서 등록 후 **\*\*Client ID\*\***와 **\*\*Client Secret\*\***을 저장해둔다. Client ID는 나중에 \`config.yml\`에 넣고, Client Secret은 Cloudflare에만 저장한다.

![cms](/assets/img/posts/screenshot-2026-04-21-at-13.56.41.png)

- - -

### **2단계: Cloudflare Worker 배포**

[sterlingwes/decap-proxy](https://github.com/sterlingwes/decap-proxy)를 클론해서 배포한다.

```bash
clone https://github.com/sterlingwes/decap-proxy
cd decap-proxycp wrangler.toml.sample wrangler.toml
```

`wrangler.toml`의 `name` 값이 워커 URL이 된다. 기본값 `decap-proxy`로 두면 `https://decap-proxy.<계정명>.workers.dev`가 된다.

이제 Cloudflare에 로그인하고 secret을 등록한 뒤 배포한다.

```bash
npx wrangler loginnpx wrangler secret put GITHUB_OAUTH_ID      # 프롬프트에 Client ID 입력
npx wrangler secret put GITHUB_OAUTH_SECRET  # 프롬프트에 Client Secret 입력
npx wrangler deploy
```

![npx](/assets/img/posts/screenshot-2026-04-21-at-14.15.44.png)

완료되면 터미널에 워커 URL이 출력된다.

```Deployed
https://decap-proxy.xxx.workers.dev
```

이 URL을 복사해두고, GitHub OAuth App의 콜백 URL을 업데이트한다.

- - -

### 3단계: admin 폴더 추가

프로젝트 루트에 `admin/` 폴더를 만들고 파일 두 개를 추가한다.

**`admin/index.html`**

```html
<!doctype html>
<html>  
<head>    
<meta charset="utf-8" />    
<meta name="viewport" content="width=device-width, initial-scale=1.0" />    
<title>Content Manager</title>  
</head>  
<body>    
<script src="https://unpkg.com/decap-cms@^3.0.0/dist/decap-cms.js"></script>  
</body>
</html>
```

**`admin/config.yml`**

```yaml
backend:
  name: github
  repo: 
  branch: main
  base_url: 

media_folder: "assets/img/posts"
public_folder: "/assets/img/posts"

collections:
  - name: "posts"
    label: "Posts"
    folder: "_posts"
    create: true
    slug: "{{year}}-{{month}}-{{day}}-{{slug}}"
    fields:
      - { label: "Title", name: "title", widget: "string" }
      - { label: "Date", name: "date", widget: "datetime" }
      - {
          label: "Categories",
          name: "categories",
          widget: "list",
          required: false
        }
      - { label: "Tags", name: "tags", widget: "list", required: false }
      - { label: "Body", name: "body", widget: "markdown" }
```

- - -

이렇게 하고 url에 /admin붙여서 들어가면, 이런 식으로 에디터를 사용할 수 있다!! 당연히 타 플랫폼의 에디터보다는 불편하지만 .md파일보다는 나은 것 같다. 

![](/assets/img/posts/screenshot-2026-04-21-at-14.28.47.png)

## **트러블슈팅**


**`Error: Failed to load config.yml (404)`**

![](/assets/img/posts/screenshot-2026-04-21-at-13.17.29.png)

### 
원인 두 가지를 확인한다.

1. `_config.yml`의 `exclude`에 `admin/config.yml`이 들어가 있는지 확인 -> 있으면 제거
2.  Chirpy 테마는 PWA 서비스워커를 사용하는데, 이게 `config.yml` 요청을 캐시에서 찾다가 404를 반환할 수 있다. 개발자도구 -> Application -> Service Workers -> Unregister 후 새로고침
