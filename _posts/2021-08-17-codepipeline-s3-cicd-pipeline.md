---
layout: post
title: "[CodePipeline] S3 CI/CD 파이프라인 구축하기 - 이론편"
summary: "CodePipeline을 이용한 S3 CI/CD 구축하기"
author: creboring
date: '2021-08-17 9:52:20 +0530'
category: AWS
thumbnail: /assets/img/posts/category/CodePipeline.png
keywords: how to use code series, frontend aws cicd, frontend code series
permalink: /blog/codepipeline-s3-cicd-pipeline/
usemathjax: true
---
## 들어가며
---
본 게시글은 주로 프런트엔드(Web)에서 많이 사용되는 정적 소스코드를 `Amazon S3` 에 **자동으로 배포**하는 과정을 다룹니다. CI/CD 구축은 프런트엔드, 백엔드 구분 없이 소스코드에 대한 빌드와 배포과정을 자동화하여 개발자로 하여금 개발에만 집중할 수 있는 환경을 제공해줍니다.

해당 자동화 방법이나 배포 대상은 매우 다양하나, 현 게시글에서는 `Amazon Web Service` 의 `Code 시리즈` 를 사용하여 **S3 정적 웹 호스팅**에 배포하는 방법을 다루고 있습니다. Code 시리즈, 또는 S3에 대해 생소하신 분들은 아래 게시글을 먼저 읽고 오실 것을 권고드립니다.
> AWS Code 시리즈가 처음이시다면 [게시글][link_1]{:target="_blank"}을 먼저 읽고 오실 것을 권해드립니다.

> AWS S3 정적 웹 호스팅에 대해 [게시글][link_2]{:target="_blank"}에 정리해두었습니다.


## 목차
---
현재 게시글은 내용이 길어져 아래와 같이 단계별로 게시글이 나뉘어져 있습니다.
- 이론: [[CodePipeline] S3 CI/CD 파이프라인 구축하기 - 이론편][link_3]{:target="_blank"}
- 실습(Source): [[CodePipeline] S3 CI/CD 파이프라인 구축하기 - Source 편][link_4]{:target="_blank"}
- 실습(Build): [[CodePipeline] S3 CI/CD 파이프라인 구축하기 - Build 편][link_5]{:target="_blank"}
- 실습(Deploy): [[CodePipeline] S3 CI/CD 파이프라인 구축하기 - Deploy 편][link_6]{:target="_blank"}


## 파이프라인 이란?
---
**파이프라인은 특정 단계의 결과물이 다음 작업의 입력(input)이 되는 구조를 뜻합니다**. 이는 CI/CD 에서만 사용되는 용어가 아닌 컴퓨터 공학에 걸쳐 전반적으로 사용되는 용어입니다. 그렇다면 CI/CD 파이프라인이란 무엇인가에 대해서도 자연스레 짐작해볼 수 있습니다. 

하나의 서비스가 **통합, 개발되어(CI), 테스트되고 배포되기(CD)** 까지, 일련의 과정을 단계별로 정리하고 이를 순차적으로 실행하는 파이프라인을 `CI/CD 파이프라인`이라 부릅니다.
- `CI(Continuous Integration)`: 직역하면 **'지속적인 통합'** 이란 뜻으로, 여러 개발자들이 함께 협업을 하며 서비스를 개발할 때, 이를 지속적으로 통합하고 중재하여 꾸준히 좋은 품질을 유지하려는 DevOps 관행 또는 그런 구조의 시스템을 의미합니다.
- `CD(Continuous Deploy/Delivery)`: 직역하면 **'지속적인 배포/전달'** 로, CI가 완료된 프로그램을 운영 환경까지 매끄럽게 배포/전달 하도록 일련의 과정들을 자동화 하는 것을 의미합니다.

<figure>
    <img src="/assets/img/posts/2021-08-17/pipeline.jpg" class="img-fluid">
    <figcaption>[<a href="http://www.freepik.com">Designed by pch.vector / Freepik</a>]</figcaption><br>
</figure>

## 파이프라인 구축 방법
---
위에서도 언급했듯이 CI/CD는 개발자로 하여금 **개발에만 집중할 수 있도록 도와주는 역할**을 합니다. 초기 구축에는 시간이 조금 소요될 수 있지만, 한번 구축이 되면 개발자가 빈번히 수행해야 했던 빌드, 배포 과정이 자동화됨으로써 많은 수동 작업들을 자동화할 수 있습니다.

각 단계마다 수행하는 작업들은 서로 다르며, **앞선 작업의 수행이 완료되어야 후행되는 작업이 수행되는 구조**를 가지고 있습니다. 보통 이런 모습을 `컨베이어 벨트` 에 비유하기도 합니다. 일반적으로 파이프라인들은 아래와 같은 절차가 존재합니다.
- Source
- Build
- Deploy
- etc.. (수동 승인, 테스트, ...)

<br>
`AWS`는 이 각각의 단계에 대해 **자체 관리형 서비스들을 제공**하고 있으며, 서비스간의 연계성이 편리해 AWS 서비스만으로도 파이프라인을 구축하는 것이 가능합니다. (물론, 각 단계마다 반드시 AWS 서비스를 사용하실 필요는 없습니다)

위 단계들을 구축하기 위해서는, 지금까지 **개발자나 운영진이 자신의 소스코드를 운영의 서버에 배포하기까지 어떠한 절차들을 거쳐왔는지 점검하고 기록할 필요가 있습니다**.

이러한 일련의 과정을 정리하고 코드화하여 파이프라인에 넘겨주게 되면, 파이프라인은 전달받은 프로세스를 하나씩 처리하며 뚝딱뚝딱 배포 작업을 수행해줄 것입니다.

<figure>
    <img src="/assets/img/posts/2021-08-17/conveyor.jpeg" class="img-fluid">
    <figcaption>[<a href="http://www.freepik.com">Designed by pch.vector / Freepik</a>]</figcaption><br>
</figure>

## 파이프라인의 아키텍쳐
---
파이프라인의 구조는 대략 아래 그림과 같이 구성됩니다. **파이프라인을 트리깅하고 동작시킬 서비스(CodePipeline)**가 존재하며, **각 파이프라인의 단계마다 역할을 수행해줄 서비스(CodeCommit, CodeBuild, ...)**가 존재합니다.

본 게시글에서 다루고 있는 정적 컨텐츠의 경우 **빌드 과정이 필수가 되지는 않습니다.** `npm`이나 별도 패키징 및 플러그인을 사용하지 않는 경우 빌드 과정을 생략하는 경우도 있으며, 빌드를 동작시키지 않아도 배포가 가능한 경우도 있습니다.

다만, 대부분의 소스코드들이 raw한 상태로 그대로 운영서버에 사용하지 않기 때문에, 소스코드에 대한 **패키징 작업이나 후처리 작업**을 빌드 과정에서 수행해줄 수 있습니다.

<img src="/assets/img/posts/2021-08-17/cicd_1.png" class="img-fluid">
- Pipeline : `CodePipeline`<br>
- Source : `CodeCommit`<br>
- Build : `CodeBuild`<br>
- Deploy : `S3`

## Source
---
Source 단계는 **CI 과정을 거쳐 배포될 소스코드를 선택하는 단계**입니다. 배포하고자 하는 소스코드가 담긴 **레포지토리**와 **브런치**를 선택해주면, 해당 브런치에 변경사항이 있을 때마다 CodePipeline을 트리깅하여 파이프라인을 동작시킵니다.

목적에 따라 서로 다른 브런치를 바라보는 것으로 **개발계**, **운영계** 파이프라인을 나눌 수도 있으며, **하나의 파이프라인에 여러 레포지토리**를 선택하여 각각의 레포지토리에서 원하는 데이터를 가져오게 할 수도 있습니다.

이처럼, Source 단계에서는 파이프라인, 서비스에 필요한 데이터가 저장된 **저장소**를 바라보는 단계입니다.<br>
이 단계에서는 CodeCommit, Github, S3 와 같은 원격 저장소들을 선택할 수 있으며, S3를 제외하고는 모두 `Git VCS(Version Control System)`에 대한 원격 저장소들이 제공되고 있습니다.

> *현재 CodePipeline은 **Git**에 대한 원격저장소는 다양하게 지원하고 있으나, 그 외의 VCS에 대해서는 지원하지 않고 있습니다. 이 경우 별도의 추가 툴이 요구될 수 있으며, 오히려 **Jenkins**와 같은 다른 Pipeline 툴을 사용하는 것이 편리하실 수도 있습니다. (작성일: 2021-09-24)*

## Build
---
Build 단계는 선택된 소스코드의 **빌드 또는 후처리 작업을 수행하는 단계**입니다. CodePipeline이 넘겨준 Source 단계의 소스코드를 지지고 볶는 단계로, raw한 소스코드를 빌드를 통해 **컴퓨터가 수행 가능한 상태로 변환**하거나, 또는 소스코드를 Deploy 하기에 앞서 **패키징이나 후처리를 통해 결과물을 다듬어주는 역할**을 수행합니다.

> Maven의 경우 `mvn package`, npm을 쓰는 경우 `npm install`, `npm run build` 를 하나의 예시로 들 수 있습니다.

Build 과정의 경우, 작업 특성상 소스코드에 대한 빌드를 직접 수행하는 단계이기 때문에, 개발자가 함께 참여하여 **빌드에 대한 커맨드를 작성해 Build 서버에 전달해주어야 합니다**. 이 단계에서는 `CodeBuild`, `Jenkins`를 사용하실 수 있으며, 필요하지 않은 경우 스킵하실 수 있습니다.

> *소스코드에 따라 빌드과정이 필요하지 않는 경우도 존재합니다. 이 경우는 CodePipeline을 생성하실 때, Build 단계를 건너뛰시고 바로 Deploy 단계를 구성하셔도 무방합니다.*


## Deploy
---
Deploy 단계는 이전 과정까지 완성된 **결과물**을 **실제 서버에 배포**해주는 역할을 수행합니다. Deploy 단계까지 도달하기 전에 소스코드는 이미 **배포 가능한 완성품 상태**여야하며, Deploy 단계는 이 결과물을 **어떤 방식**으로 **어떤 서버**의 **어떤 위치**에 배포할 것인지를 결정해야 합니다.

배포 대상과 방법은 매우 다양합니다(Blue Green, Rolling, ...). 그 중 저희가 결과물을 어떤 서버에 배포하는지에 따라 선택할 수 있는 방법의 폭이 달라지게 되며, 몇몇 서비스들은 배포방법을 지정하지 않아도 대상 서비스에서 자체적으로 배포를 수행합니다.



## 실제 구축해보기
---
실습은 위 **'구축 방법'** 에서 설명드린 순서대로 진행됩니다. 먼저, 소스코드를 저장할 ***레포지토리*** 를 만들어 코드를 커밋하고, ***빌드 서버*** 를 추가하여 후처리 작업을 수행할 수 있도록 ***스크립트*** 를 넘겨줄 예정입니다. 마지막으로 결과물을 ***S3*** 에 배포하도록 파이프라인에 지정하면 끝입니다. 

이 모든 과정은 **CodePipeine**에 의해 트리깅되며, CodePipeline은 그 실행 결과를 사용자에게 보여줍니다. 이제 실제로 AWS Console에 접속하여, 실습을 진행해보겠습니다 :)
> AWS 콘솔은 주기적으로 많이 업데이트되고 변경됩니다. 단순히 화면을 따라하시기 보다는, 내용을 이해하시고 이에 부합한 버튼과 입력을 해주시면 되겠습니다.

- 다음: [[CodePipeline] S3 CI/CD 파이프라인 구축하기 - Source 편][link_4]{:target="_blank"}


[link_1]: https://creboring.github.io/blog/what-is-code-series/
[link_2]: https://creboring.github.io/blog/what-is-s3-static-web-hosting/
[link_3]: https://creboring.github.io/blog/codepipeline-s3-cicd-pipeline/
[link_4]: https://creboring.github.io/blog/codepipeline-s3-cicd-pipeline-2/
[link_5]: https://creboring.github.io/blog/codepipeline-s3-cicd-pipeline-3/
[link_6]: https://creboring.github.io/blog/codepipeline-s3-cicd-pipeline-4/