---
layout: post
title: "[Linux] jq로 느낌 있게 JSON 데이터 가공하기"
summary: "jq를 통해 command line 에서 빠르게 JSON 가공하기"
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
엔지니어로 여러 업무들을 수행하다보면 **JSON 문자열**과 자주 만나게 됩니다. <br>(서버 목록을 뽑는다거나, Request Parameter를 분석한다거나...) 

특히나, cli 커맨드를 통해 주고받는 데이터를 원하는 포맷으로 보고 싶을 때가 많은데, 간단한 스크립트 언어를 통해 구현해도 되고 텍스트 에디터를 통해 수정해도 되지만 간단한 리스팅을 할 때도 이렇게 수고를 들이는건 너무 귀찮습니다.

이럴 때 cli를 통해 쉽고 가볍게 JSON을 핸들링(필터링)할 수 있는 jq라는 툴이 존재하는데요. 최근들어 유용하게 사용하고 있는 `jq` 에 대해 분석한 내용과 간단한 사용 방법을 공유드리겠습니다.

## jq란?

<figure>
    <img src="/assets/img/posts/2024-01-23/jq_explain.gif" class="img-fluid">
    <figcaption><small>JSON object의 name만 필터링 하는 예시</small></figcaption>
</figure>

jq는 전달받은 JSON 데이터를 손쉽게 접근하고 가공할 수 있는 light weight 프로세서입니다. 간단한 문법을 통해 원하는 데이터를 검색할 수 있고, 필터링할 수 있으며 이를 재가공해서 다른 포맷으로 변환할 수도 있습니다. (제공되는 기능이 정말 많습니다)

사실은 jq가 있기 이전, 이와 비슷한 기능을 제공하는 툴들은 많이 존재해왔습니다. jq의 영감이 되기도 한 sed 도 그렇고, sed, awk, grep 모두 CLI에서 텍스트를 편하게 핸들링하기 위한 수단으로써 개발되어 왔습니다. 
jq 또한 마찬가지로 '텍스트' 그 중에서도 JSON 데이터를 쉽게 다루자는 의의에서 개발되었으며, 여러 OS를 지원하기 때문에 맞는 버전만 설치하면 어디서든 동일한 문법으로 사용할 수 있습니다 😎 (약간의 차이는 있습니다)

> **"jq is like sed for JSON data"** <br> - jq project document

아래는 jq를 통해 JSON 데이터를 가공한 간단한 예시입니다. 역시 설명보단 실제로 사용해보는 것이 빠르기 때문에, 바로 설치 방법부터 간단한 문법까지 맛보여드리겠습니다.

```
$ cat test-data.JSON | jq '.["name"].[]' 
```

## 설치 방법
jq는 많은 패키지 매니저들에서 기본적으로 제공되는 라이브러리 중 하나 입니다. 각 OS마다 지원되는 패키지 매니저를 통해 손쉽게 다운받을 수 있는데, 예시로 MacOS에서는 homebrew를 통해 아래와 같이 설치할 수 있습니다.

```sh
$ brew install jq
```

윈도우의 경우도 winget 또는 chocolatey를 통해 아래와 같이 설치가 가능합니다.

```sh
# $ winget install jqlang.jq
$ chocolatey install jq
```

이 외에도 각 OS마다 설치법이 공식 문서에 안내되어 있으니 참고하셔서 다운 받으시면 됩니다.
> <a href="https://jqlang.github.io/jq/download/" target="_blank">jq Document (how to download)</a>

## 문법

<figure>
    <img src="/assets/img/posts/2024-01-23/jq_docs.gif" class="img-fluid">
    <figcaption><small>저게 다 필터</small></figcaption>
</figure>

jq 문법은 **기본 구조만 이해하시면** 나머지는 키워드를 통해 필요한 필터들을 찾아가며 활용해가면 됩니다. 
필터들이 문서화가 정말 잘 되어 있기 때문에 문서상의 사용 예시를 참고하시면 되며, 그럼에도 이해가 안될 시 해당 문법을 테스트해볼 수 있는 사이트도 제공되고 있습니다.<br>
(~~문법도 많아서 공식 문서에서 문법 찾는게 일이에요.~~)

아래 예시는 맛보기 정도로 보시고, 어떻게 사용될 수 있는지 정도로 이해해주시면 좋겠습니다. <br>자세한건 공식 문서 찾아보세요 😉

> <a href="https://jqlang.github.io/jq/manual/" target="_blank">jq 공식 문서</a>


### 기본 구조
jq는 JSON 데이터 스트림에 jq 문법을 활용한 쿼리를 문자열로 넘겨주며 사용합니다. 
```sh
# 파이프 문법을 통해 jq 커맨드의 input으로 test.json 파일의 데이터 스트림을 넘겨줌
$ cat test.json | jq 
```
쿼리의 시작은 항상 `.` 으로 시작되며, 이 점은 하나의 개체(identity)를 의미합니다. 이 데이터는 JSON 데이터라는 것을 알려주는 것이죠.

처음 `.`이 사용되면 input으로 넘겨준 JSON의 최상단 위치를 가리키고 있습니다. 이후 한 계단씩 원하는 데이터를 찾을 때까지 drill down하며 문법을 작성해나가면 됩니다.

```sh
$ cat test.json | jq '.' # 현재 개체를 선택만 한 상태 (input, output 같음)
```

이후부터는 데이터의 구조를 보며 원하는 값들을 찾아내면 되는데, 이는 각 데이터의 타입에 따라 다릅니다.

#### Object
가장 많이 보게 되는 형태로, Object의 Key 이름을 `.`의 다음 값에 명시해주면 됩니다. 그 방법은 총 2가지 인데, 가장 기본적인 방식인 `["key"]` 형식의 대괄호를 활용한 방식이 있으며, 대괄호와 쌍따음표를 생략하고 바로 키 값을 넘겨주는 방식이 있습니다.
 아래 예시를 통해 살펴보겠습니다.

```sh
# test.json 파일 내용 
{
    "name": "dohun lee",
    "description": "대략적인 설명",
    "role": "devops engineer",
    "address": {
        "country": "Korea",
        "city": "Seoul",
        "street address": "일급 비밀"
    },
    "pet": [
        {
            "name": "sogum",
            "type": "cat",
            "age": 5
        },
        {
            "name": "yamato",
            "type": "shrimp",
            "age": 2
        }
    ]
}
```

위와 같은 데이터에서 만약 role 정보를 알고 싶다면 아래와 같이 입력하면 됩니다.

```sh
# 아래 두 커맨드는 동일한 커맨드
$ cat test.json | jq '.["role"]' 
$ cat test.json | jq '.role'
```

위 문법에서 `.["role"]` 는 `.role` 과 같이 생략되어 사용될 수 있습니다.
(다만, 키 값에 공백이 포함되는 경우 문법상 에러가 날 수 있으니 그런 경우는 대괄호를 사용해 키 값을 명확히 명시해줍시다.)

```sh
# output
"devops engineer"
```

그런데 위 Object의 `address` 데이터와 같이 데이터의 반환값이 문자열이 아닌 또 다른 Object라면? 마찬가지로 해당 데이터의 개체를 선택해주고, 다시 키 값을 기반으로 탐색해 나가면 됩니다.

```sh
# 아래 두 커맨드는 같은 커맨드 
$ cat test.json | jq '.["address"].["street address"]' 
$ cat test.json | jq '.address["street address"]' 
```

```sh
# output
"일급 비밀"
```

> 참고로, `.` 문법은 최초에만 사용하면 되며, `[]` 대괄호를 선택자 문법으로 사용하는 경우 `.`은 생략될 수 있습니다. (대괄호를 생략하는 경우에는 `.` 필수)

#### Array
Array의 경우 Array 전체를 탐색하며 반복문을 수행하는 방법이 있고, 원하는 index를 선택해 탐색하는 방법이 있습니다.

먼저 index를 이용해 하나의 데이터에 접근할 때는 배열의 index를 기존 선택자 뒤에 대괄호로 명시해주면 됩니다. (Array의 경우 대괄호 생략이 불가합니다.)

```sh
$ cat test.json | jq '.pet[0]' 
```

```sh
# output
{
    "name": "sogum",
    "type": "cat",
    "age": 5
}
```
마찬가지로, 반환 값이 Object라면 선택자로 key 정보를 넘겨줘 원하는 값만 필터링할 수 있습니다.

```sh
$ cat test.json | jq '.pet[0].name' 
```

```sh
# output
"sogum"
```

만약 배열의 전체를 반복문을 돌며 필터링하거나 접근하고 싶은 경우, index를 명시하지 않고 전체를 호출하거나, 특정 index 구간을 선택해 원하는 데이터들만 호출할 수 있습니다. 방법은 마찬가지로 대괄호를 사용하면 되는데, 아래와 같이 대괄호 안에 아무런 값을 명시하지 않으면 배열 전체가 반복문을 돌며 호출됩니다.
```sh
$ cat test.json | jq '.pet[]' 
```

```sh
# output
{
    "name": "sogum",
    "type": "cat",
    "age": 5
}
{
    "name": "yamato",
    "type": "shrimp",
    "age": 2
}
```

배열의 각 개체들에서 name만 추출하고 싶다면 아래와 같이 사용하면 됩니다.
```sh
$ cat test.json | jq '.pet[].name' 
```

```sh
# output
"sogum"
"yamato"
```

### 필터 파이프라이닝
마지막으로 하나만 더 맛본다고 한다면, 필터 파이프라인은 꼭 드셔보셨으면 합니다 🤵🏻‍♂️

위에서 보여드린건 각 JSON 데이터들에 접근하는 기본 방식이고 아래는 데이터를 필터링할 때 자주 쓰이는 파이프라인 예제들입니다. jq 내에서 사용되는 파이프 `|` 문법도 Linux에서 사용되는 문법과 동일합니다. 앞선 jq 문법에서의 output을 다시 jq 문법의 input으로 쓰거나, 또는 필터 문법을 사용해 앞선 jq 결과물을 필터링할 수 있습니다.

## jq 동작 원리

## 마무리