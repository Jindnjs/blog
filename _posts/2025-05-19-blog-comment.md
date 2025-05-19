---
title: "[Blog] Giscus 댓글 기능 추가하기"
excerpt: "Github 블로그에 댓글 기능을 추가해보자. 여러가지 댓글 플랫폼을 비교하고 적용해보자"

categories: # 카테고리 설정
  - Blog
tags: # 포스트 태그
  - [Blog]

permalink: /blog/comment/ # 포스트 URL

toc: true # 우측에 본문 목차 네비게이션 생성
toc_sticky: true # 본문 목차 네비게이션 고정 여부

date: 2025-05-19 # 작성 날짜
last_modified_at: 2025-05-19 # 최종 수정 날짜
comments: true
---

---

## 들어가며

## 플랫폼 비교

### utterances

### didqus

### giscus

### 요약

## 적용하기

--- 

### 1. Giscus App 설치

먼저 [링크](https://github.com/apps/giscus)에서 Giscus App를 설치한다. 

<img width="1028" alt="Image" src="https://github.com/user-attachments/assets/0ca967f0-322c-4dcb-80e8-c313c7d088dd" />
<img width="593" alt="Image" src="https://github.com/user-attachments/assets/a353651a-ca2c-4de6-b548-56c8657aac95" />

댓글을 적용하고 싶은 대상 Repository를 선택하고, Install 버튼을 클릭한다.

---

### 2. Repository - Discussions 활성화

Giscus는 Github의 Discussions기반으로 동작하여 Discussions를 활성화 해야한다.  
Repository에 App을 설치한다음, Discussion을 활성화(체크)한다.  
Discussion은 `Settings` > `General` > `Features`에서 설정한다.

---

### 3. Discussions 설정

Discussions에서 댓글을 위한 Category를 설정한다.  
`Discussions` > `New category`에서 Comments를 만든다.

아래와 같이 설정한 후, `Create`를 눌러 생성한다.

<img width="781" alt="Image" src="https://github.com/user-attachments/assets/e090c616-2da0-4f55-b3ea-d058eac573ae" />

사용하지 않는 Category는 정리해도 무방하다.

---

### 4. Giscus 설정

다시 [링크](https://giscus.app/ko)에 접속하여 `Giscus` 설정을 한다.  
우선 `저장소`에 블로그 Repository를 입력한다. 입력하기 전에 아래 조건을 만족하는지 다시 확인해보자.

> 1. 공개 저장소여야 합니다. 그렇지 않으면 방문자들은 Discussion을 볼 수 없습니다.
> 2. giscus 앱이 설치되어 있어야 합니다. 그렇지 않으면 방문자들은 댓글과 반응을 남길 수 없습니다.
> 3. Discussions 기능이 해당 저장소에서 활성화되어 있어야 합니다.

<img width="600" alt="Image" src="https://github.com/user-attachments/assets/ca8fb479-dec1-4e85-9a23-a3c8e9798791" />

---

Giscus와 Discussions를 연결하는 방법을 선택한다.  
옵션과 설명을 읽고, 상황에 맞는 연결법을 선택하면 된다.  
보통 첫번째 옵션을 많이 사용한다.  
Discussion Category는 앞서 만든 Comments를 선택한다.

선택 완료후, 아래 `giscus 사용`에 script 코드가 표시된다.

<img width="790" alt="Image" src="https://github.com/user-attachments/assets/fbad3591-2cf8-45ba-a1ee-2d334dc1d29d" />

해당 코드를 원하는 위치에 적용하면 된다.

---

### 5. 적용하기

Jekyll 테마 기반의 페이지는 `_config.yml` 파일에 giscus 설정이 가능하다.  
위에서 생성된 script 코드에 맞는 내용들을 `_config.yml`에 옮겨 적으면 적용이된다

```yml
...
comments:
  provider               : "giscus"
  disqus:
    shortname            : # https://help.disqus.com/customer/portal/articles/466208-what-s-a-shortname-
  discourse:
    server               : # https://meta.discourse.org/t/embedding-discourse-comments-via-javascript/31963 , e.g.: meta.discourse.org
  facebook:
    # https://developers.facebook.com/docs/plugins/comments
    appid                :
    num_posts            : # 5 (default)
    colorscheme          : # "light" (default), "dark"
  utterances:
    theme                : # "github-light" (default), "github-dark"
    issue_term           : # "pathname" (default)
  giscus:
    repo_id              : # Shown during giscus setup at https://giscus.app
    category_name        : # Full text name of the category
    category_id          : # Shown during giscus setup at https://giscus.app
    discussion_term      : # "pathname" (default), "url", "title", "og:title"
    reactions_enabled    : # '1' for enabled (default), '0' for disabled
    theme                : # "light" (default), "dark", "dark_dimmed", "transparent_dark", "preferred_color_scheme"
    strict               : # 1 for enabled, 0 for disabled (default)
    input_position       : # "top", "bottom" # The comment input box will be placed above or below the comments
    emit_metadata        : # 1 for enabled, 0 for disabled (default) # https://github.com/giscus/giscus/blob/main/ADVANCED-USAGE.md#imetadatamessage
    lang                 : # "en" (default)
    lazy                 : # true, false # Loading of the comments will be deferred until the user scrolls near the comments container.
...
```

---

## 결론 

오늘은 블로그에 Giscus를 활용하여 댓글 기능을 추가하였다.  
몇가지의 댓글 서비스를 분석해보니, Giscus가 가장 적합하다고 생각하였다.  
[공식 문서](https://giscus.app/ko)도 한국어로 잘 정리되어있어 참고하는데 도움이 되었다.

---

## 덧붙이며

혹시 궁금한 점이 있다면 댓글로 남겨 주셔도 좋습니다.

---
