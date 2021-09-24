---
layout: post
title: "[CodePipeline] S3 Static Website에 CI/CD 파이프라인 구축하기"
summary: "CodePipeline을 이용한 S3 CI/CD 구축하기"
author: creboring
date: '2021-08-17 9:52:20 +0530'
category: AWS
thumbnail: /assets/img/posts/CodePipeline.png
keywords: how to use code series, frontend aws cicd, frontend code series
permalink: /blog/codepipeline-frontend-source-cicd/
usemathjax: true
---
## ✔️ 개요
---
본 게시글은 프런트엔드에서 사용되는 정적 소스코드를 ***S3 정적 웹 호스팅***에 **자동**으로 **배포**하는 과정을 다룹니다.

CI/CD 구축은 프런트엔드, 백엔드 구분 없이 소스코드에 대한 빌드와 배포과정을 자동화하여 개발자로 하여금 개발에만 집중할 수 있는 환경을 제공해줍니다.

해당 자동화 방법이나 배포 대상은 매우 다양하나, 현 게시글에서는 ***Amazon Web Service*** 의 **Code 시리즈**를 사용하여 **S3 정적 웹 호스팅**에 배포하는 방법을 다루고 있습니다.

Code 시리즈, 또는 S3에 대해 생소하신 분들은 아래 게시글을 먼저 읽고 오실 것을 권고드립니다.
> AWS Code 시리즈란?
[게시글 읽기][link_1]{:target="_blank"}

> AWS S3 정적 웹 호스팅이란?
[게시글 읽기][link_2]{:target="_blank"}


<br>
## ⚙️ 아키텍쳐
---
<img src="/assets/img/posts/2021-08-17-codepipeline-frontend-source-cicd_1.png" class="img-fluid">
- Pipeline : **CodePipeline**<br>
- Source : **CodeCommit**<br>
- Build : **CodeBuild**<br>
- Deploy : **S3**

<br>
정적 컨텐츠의 경우 **빌드 과정이 필수가 되지는 않습니다.**<br>
npm이나 별도 패키징 및 플러그인을 사용하지 않는 경우 빌드 과정을 생략하는 경우도 있으며, 빌드를 동작시키지 않아도 배포가 가능한 경우도 있습니다.

다만, 대부분의 소스코드들이 raw한 소스코드를 그대로 운영서버에서 사용하지 않기 때문에, 소스코드에 대한 **패키징 작업**이나 **후처리 작업**을 빌드 과정에서 수행해줄 수 있습니다.


<br>
## 🪜 구축 방법
---
위에서도 언급했듯이 CI/CD는 개발자로 하여금 개발에만 집중할 수 있도록 도와주는 역할을 합니다.<br>
실제 구축에 앞서, 아래 각 CI, CD 단계의 구축이 어떤 방향으로 진행되는지 알아보겠습니다.

먼저, CodePipeline은 아래와 같은 단계가 존재합니다.
- Source
- Build
- Deploy
- etc.. (수동 승인, 테스트, ...)

이 중에서도 단연 중요한 단계는 **Source, Build, Deploy** 단계인데, 각 단계는 서로 **분리된 목적**으로 각각의 역할을 수행하며, AWS는 이 각각의 단계에 대해 자체 관리형 서비스들을 제공하고 있습니다. (물론, 각 단계마다 반드시 AWS 서비스를 사용하실 필요는 없습니다)

그럼, 각 단계에서 수행하는 역할과, AWS에서 제공하는 서비스에 대해 알아보겠습니다.

<br>
#### **Source**
\\\* *AWS 서비스: CodeCommit* \*\\

Source 단계는 **CI 과정을 거쳐 배포될 소스코드를 선택하는 단계**입니다.<br>
배포하고자 하는 소스코드가 담긴 **레포지토리**와 **브런치**를 선택해주면, 해당 브런치에 변경사항이 있을 때마다 CodePipeline을 트리깅하여 파이프라인을 동작시킵니다.

목적에 따라 서로 다른 브런치를 바라보는 것으로 **개발계**, **운영계** 파이프라인을 나눌 수도 있으며, **하나의 파이프라인에 여러 레포지토리**를 선택하여 각각의 레포지토리에서 원하는 데이터를 가져오게 할 수도 있습니다.

이처럼, Source 단계에서는 파이프라인, 서비스에 필요한 데이터가 저장된 **저장소**를 바라보는 단계입니다.<br>
이 단계에서는 CodeCommit, Github, S3 와 같은 원격 저장소들을 선택할 수 있으며, S3를 제외하고는 모두 **Git VCS(Version Control System)**에 대한 원격 저장소들이 제공되고 있습니다.

> *※ 현재 CodePipeline은 Git에 대한 원격저장소는 다양하게 지원하고 있으나, 그 외의 VCS에 대해서는 지원하지 않고 있습니다. 이 경우 별도의 추가 툴이 요구될 수 있으며, 오히려 Jenkins와 같은 다른 Pipeline 툴을 사용하시는 것이 편리하실 수도 있습니다. (작성일: 2021-09-24)*


<br>
#### **Build**
\\\* *AWS 서비스: CodeBuild* \*\\

Build 단계는 선택된 소스코드의 **빌드**, 또는 **후처리 작업**을 수행하는 단계입니다.<br>
CodePipeline이 넘겨준 Source 단계의 소스코드를 지지고 볶는 단계로, raw한 소스코드를 빌드를 통해 **컴퓨터가 수행 가능한 상태로 변환**하거나, 또는 소스코드를 Deploy 하기에 앞서 **패키징**이나 **후처리**를 통해 결과물을 다듬어주는 역할을 수행합니다.

Java의 경우 `mvn package`, node.js를 쓰는 경우 `npm install`, `npm run build` 를 하나의 예시로 들 수 있습니다.

Build 과정의 경우, 작업 특성상 소스코드에 대한 빌드를 직접 수행하는 단계이기 때문에, 개발자가 함께 참여하여 빌드에 대한 커맨드를 작성해 Build 서버에 전달해주어야 합니다. (CodeBuild의 경우 ***buildspec.yaml*** 이라는 파일을 사용합니다.)

아래는 현 게시글의 실습단계에서 사용할 *buildspec.yaml* 파일입니다.
``` yaml
version: 0.2

phases:
  install:
    runtime-versions:
      nodejs: 14.x
  pre_build:
    commands:
      - npm install
  build:
    commands:
      - npm run build
artifacts:
  base-directory: 'dist'
  files:
    - '**/*'
```

이 단계에서는 CodeBuild, Jenkins를 사용하실 수 있으며, 필요하지 않은 경우 스킵하실 수 있습니다.

> *소스코드에 따라 빌드과정이 필요하지 않는 경우도 존재합니다. 이 경우는 CodePipeline을 생성하실 때, Build 단계를 건너뛰시고 바로 Deploy 단계를 구성하셔도 무방합니다.*


<br>
#### **Deploy**
\\\* *AWS 서비스: CodeDeploy* \*\\

Deploy 단계는 이전 과정까지 완성된 **결과물**을 **실제 서버에 배포**해주는 역할을 수행합니다.<br>
Deploy 단계까지 도달하기 전에 소스코드는 이미 **배포 가능한 완성품 상태**여야하며, Deploy 단계는 이 결과물을 **어떤 서버**의 **어떤 위치**에, **어떤 방식**으로 배포할 것인지를 결정해야 합니다.

배포 대상과 방법은 매우 다양합니다(Blue Green, Rolling, ...). 그 중 저희가 결과물을 어떤 서버에 배포하는지에 따라 선택할 수 있는 방법의 폭이 달라지게 되며, 몇몇 서비스들은 배포방법을 지정하지 않아도 대상 서비스에서 자체적으로 배포를 수행합니다.



<br>
## 💻 실습
---
실습은 아래와 같은 순서로 진행됩니다.

<br>
## ☑️ 끝으로
---
test

[link_1]: https://creboring.github.io/blog/what-is-code-series/
[link_2]: https://creboring.github.io/blog/what-is-s3-static-web-hosting/
