---
layout: post
title: "[Linux] jq로 느낌 있게 json 데이터 가공하기"
summary: "jq를 통해 command line 에서 빠르게 json 가공하기"
author: creboring
date: '2024-01-23 23:33:50 +0900'
category: [Linux]
thumbnail: /assets/img/posts/2024-01-23/thumbnail.jpg
title-img: /assets/img/posts/category/blacksmith.jpg
keywords: Terraform, 테라폼, terraform cloud, cloud, aws, 테라폼 클라우드, 클라우드
permalink: /blog/how-to-use-jq/
usemathjax: true
---

<figure>
    <img src="/assets/img/posts/2024-01-23/thumbnail.jpg" class="img-fluid">
    <figcaption><small>{"인트로": "Hello, World"}</small></figcaption>
</figure>

<!-- excerpt-start -->
엔지니어로 여러 업무들을 수행하다보면 **json 문자열**과 자주 만나게 됩니다. <br>(서버 목록을 뽑는다거나, Request Parameter를 분석한다거나...) 

특히나, cli 커맨드를 통해 주고받는 데이터를 원하는 포맷으로 보고 싶을 때가 많은데, 간단한 스크립트 언어를 통해 구현해도 되고 텍스트 에디터를 통해 수정해도 되지만, 간단한 리스팅을 할 때도 이런 작업을 하는 것은 너무 공수가 많이 듭니다.

이와 같은 고민은 과거부터 이어져왔고 cli의 파이프 기능을 통해 쉽고 가볍게 json을 핸들링하자! 라는 목적을 가진 jq가 개발되었습니다. 최근들어 유용하게 사용하고 있는 `jq` 에 대해 분석한 내용을 공유드리겠습니다.

## jq란?
jq는 전달받은 JSON 데이터를 손쉽게 접근하고 가공할 수 있는 light weight 프로세서입니다. 간단한 문법을 통해 원하는 데이터를 검색할 수 있고, 필터링할 수 있으며 이를 재가공해서 다른 포맷으로 변환할 수도 있습니다. (제공되는 기능이 정말 많습니다)

사실은 jq가 있기 이전, 이와 비슷한 기능을 제공하는 툴들은 많이 존재해왔습니다. jq의 영감이 되기도 한 sed 도 그렇고, sed, awk, grep 모두 CLI에서 텍스트를 편하게 핸들링하기 위한 수단으로써 개발되어 왔습니다. 
jq 또한 마찬가지로 '텍스트' 그 중에서도 json 데이터를 쉽게 다루자는 의의에서 개발되었으며, 여러 OS를 지원하기 때문에 맞는 버전만 설치하면 어디서든 동일한 문법으로 사용할 수 있습니다 😎

> **"jq is like sed for JSON data"** <br> - jq project document

실제로 jq에 익숙해지신다면 스크립트 작성 없이 json 내 몇백개의 데이터를 필터링하고 가공하여 산출물까지 만들 수 있는 스킬을 습득하실 수 있습니다. (저는 안되지만, 여러분은 간단한 요구사항은 그 자리에서 슥삭 해결하실 수도 있습니다)

아래는 jq를 통해 json 데이터를 가공한 간단한 예시입니다. 해당 문법과 원리에 대해 분석한 내용을 아래에서 자세히 공유하겠습니다.

```
$ cat test-data.json \
    | jq '.["name"].[]' 
```

## jq 문법
jq는 문서화가 정말 잘되어 있습니다...<br>
(~~문법도 많아서 공식 문서에서 문법 찾는게 일이에요.~~)

> <a href="https://jqlang.github.io/jq/manual/" target="_blank">https://jqlang.github.io/jq/manual/</a><br> - jq 공식 문서

별도 글이나 stackoverflow도 필요 없을만큼 예시까지 포함된 상세한 공식 문서를 제공합니다. 제 글에서는 이 문법에 대해 조금 더 쉽게 이해하실 수 있게 단계적으로 구조를 설명드릴 예정입니다. 이후 추가적으로 커맨드들을 익히고 싶으시다면 위 공식 문서를 찾아보실 것을 추천드립니다.


### 기본 구조

### Object 처리

### Array 처리

### 필터링

## jq 동작 원리

## 마무리