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

블로그에 글을 작성하며, 블로그 방문자와 소통이 필요하다고 생각했다.  
단순히 정보를 전달하는것을 넘어, 질문과 의견을 주고 받는 과정에서 더 좋은 글이 나온다 생각한다.
이러한 이유로 블로그에 댓글 기능을 도입하고자 하였고, 몇가지의 댓글 서비스들을 비교한 결과 Giscus를 선택하였다.

이번 글에서는 몇가지 댓글 서비스 비교 및 댓글 기능 적용 과정을 정리해보고자 한다.

---

## 플랫폼 비교

깃허브 블로그를 위한 댓글 서비스는 여러가지 존재한다.  
이중, 많은 깃허브 블로그 작성자들이 주로 사용하는 서비스는 disqus, utterances, giscus 3가지의 서비스를 비교해보았다.

---

### Disqus

[Disqus](https://disqus.com/)는 다양한 웹사이트와 플랫폼에서 사용가능한 외부 댓글 서비스이다.
설치와 적용이 간편하며, 댓글은 다양한 소셜 로그인을 통해 댓글을 작성한다는 특징이 있다.  
하지만 많은 사용자들이 공통적으로 말하는 단점으로는 광고가 많다는점이 있다.

<figure>
	<a href="/assets/images/blog-comment/1.jpg"><img src="/assets/images/blog-comment/1.jpg"></a>
    <figcaption>Disqus 적용모습</figcaption>
</figure>

{% capture notice-1 %}
#### New Site Features

* You can now have cover images on blog pages
* Drafts will now auto-save while writing
{% endcapture %}

<div class="notice">{{ notice-1 | markdownify }}</div>

{% capture notice-2 %}
#### New Site Features

* You can now have cover images on blog pages
* Drafts will now auto-save while writing
  {% endcapture %}

<div class="notice--danger">{{ notice-2 | markdownify }}</div>

<h4> 특징 </h4>

> - 소셜 로그인을 통해 댓글 작성이 가능하다.
> - 무료 광고가 삽입되어 가독성이 좋지않다.
> - 댓글 및 대댓글 기능이 지원되어 댓글 피드백이 가능하다.
> - Disqus서버에 댓글이 저장된다.

---

### Utterances

[Utterances](https://utteranc.es/)
![Image](https://github.com/user-attachments/assets/593b6c5b-5102-450d-83c9-ef769f68e102)

Utterances는 Github의 Issues를 댓글로 활용하는 댓글 서비스이다.  
댓글이 특정 Github저장소에 저장되며, 포스트 하나당 이슈 하나가 연결된다. Github 기반으로 동작하기 때문에 댓글 사용을 위해 Github로그인이 필요하다.

<h4> 특징 </h4>

> - Github 로그인을 통해 댓글 작성이 가능하다.
> - 댓글은 GitHub Issue에 저장되며, 마크다운/이모지/알림 기능 등을 그대로 쓸 수 있다.
> - 마크다운 언어를 지원한다.
> - 대댓글 기능이 없다.
> - 광고가 없다.

---

### Giscus

[Giscus](https://giscus.app)
![Image](https://github.com/user-attachments/assets/93debd7e-53cb-4715-af10-b1e3f478b404)

Giscus는 Github의 Discussions 기반으로 작동하는 댓글 서비스이다.  
댓글이 Github - Discussion에 남는 특징이 있으며, 이로인해 별도 서버없이 사용 가능하며, 광고가 없는 장점이 있다. Utterances와 마찬가지로 Github 로그인을 통해 사용가능하다.

<h4> 특징 </h4>

> - Github 로그인을 통해 댓글 작성이 가능하다.
> - Github Discussions 기반으로 작동한다.
> - 댓글 및 대댓글 기능이 지원되어 댓글 피드백이 가능하다.
> - 광고가 없다.

---

### 요약

| 항목     | Discus     | Utterances    | Giscus             |
|--------|------------|---------------|--------------------|
| 저장위치   | 자체 플랫폼     | Github Issues | Github Discussions |
| 로그인 방식 | 익명 & 소셜로그인 | Github 계정     | Github 계정          |
| 광고     | 있음         | 없음            | 없음                 |
| 마크다운   | 제한         | 가능            | 가능                 |
| 대댓글    | 가능         | 불가능           | 가능                 |

---

## 적용하기

세가지 서비스를 비교분석해본 결과 Giscus를 적용하는것이 가장 적합하다고 판단하였다.  

우선 Discus는 로그인 방식이 소셜 로그인이기 때문에, 개발 블로그의 특성을 생각해보면 적합하지 않다 생각된다. 방문자 대부분이 Github사용자일 것이고, 소셜 로그인은 아무래도 개인 소셜 계정을 노출시켜야 한다는점을 고려해보면 Discus보다 Giscus가 적합하다 생각된다.  

또한, Utterances와 Giscus는 둘다 Github 기반이지만, Utterances는 대댓글이 제한되는 점에서, 방문자와의 상호작용은 Giscus가 더 유리하다고 생각된다.

아래에서는 Giscus를 블로그에 적용하는 과정을 정리해보았다.

--- 

### 1. Giscus App 설치

먼저 [링크](https://github.com/apps/giscus)에서 Giscus App를 설치한다. 

![Image](https://github.com/user-attachments/assets/0ca967f0-322c-4dcb-80e8-c313c7d088dd)
![Image](https://github.com/user-attachments/assets/a353651a-ca2c-4de6-b548-56c8657aac95)

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

![Image](https://github.com/user-attachments/assets/e090c616-2da0-4f55-b3ea-d058eac573ae)

사용하지 않는 Category는 정리해도 무방하다.

---

### 4. Giscus 설정

다시 [링크](https://giscus.app/ko)에 접속하여 `Giscus` 설정을 한다.  
우선 `저장소`에 블로그 Repository를 입력한다. 입력하기 전에 아래 조건을 만족하는지 다시 확인해보자.

> 1. 공개 저장소여야 합니다. 그렇지 않으면 방문자들은 Discussion을 볼 수 없습니다.
> 2. giscus 앱이 설치되어 있어야 합니다. 그렇지 않으면 방문자들은 댓글과 반응을 남길 수 없습니다.
> 3. Discussions 기능이 해당 저장소에서 활성화되어 있어야 합니다.

![Image](https://github.com/user-attachments/assets/ca8fb479-dec1-4e85-9a23-a3c8e9798791)

---

Giscus와 Discussions를 연결하는 방법을 선택한다.  
옵션과 설명을 읽고, 상황에 맞는 연결법을 선택하면 된다.  
보통 첫번째 옵션을 많이 사용한다.  
Discussion Category는 앞서 만든 Comments를 선택한다.

선택 완료후, 아래 `giscus 사용`에 script 코드가 표시된다.

![Image](https://github.com/user-attachments/assets/fbad3591-2cf8-45ba-a1ee-2d334dc1d29d)

해당 코드를 원하는 위치에 적용하면 된다.

---

### 5. 적용하기

Jekyll 테마 기반의 페이지는 `_config.yml` 파일에 giscus 설정이 가능하다.  
위에서 생성된 script 코드에 맞는 내용들을 `_config.yml`에 옮겨 적으면 적용된다.

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
