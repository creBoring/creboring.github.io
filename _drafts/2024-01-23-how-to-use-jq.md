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
    <img src="/assets/img/posts/2024-01-23/frontend-backend-json.jpg" class="img-fluid">
    <figcaption><small>{"인트로": "Hello, World"}</small></figcaption>
</figure>

<!-- excerpt-start -->
엔지니어로 여러 업무들을 수행하다보면 **json 문자열**과 자주 만나게 됩니다. <br>(서버 목록을 뽑는다거나, Request Parameter를 분석한다거나...) 

특히나, cli 커맨드를 통해 주고받는 데이터를 원하는 포맷으로 보고 싶을 때가 많은데, 이를 간단한 스크립트 언어를 통해 구현해도 되고 텍스트 에디터를 통해 수정해도 되지만, 간단한 리스팅을 할 때도 이런 작업을 하는 것은 너무 공수가 많이 듭니다.

이와 같은 고민은 과거부터 이어져왔고 cli의 파이프 기능을 통해 쉽고 가볍게 json을 핸들링하자! 라는 목적을 가진 jq가 개발되었습니다. 최근들어 유용하게 사용하고 있는 `jq` 에 대해 분석한 내용을 공유드리겠습니다.
> jq Github Repo: https://github.com/jqlang/jq

## jq란?