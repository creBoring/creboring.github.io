---
layout: post
title: "CodePipeline으로 배포 파이프라인 구축하기"
summary: "강조되고 반복되는 배포는 개발자를 불안하게 해요!"
author: creboring
date: '2022-09-27 9:52:20 +0530'
category: AWS
thumbnail: /assets/img/posts/category/CodePipeline.png
title-img: /assets/img/posts/category/CodePipeline.png
keywords: how to use code series, aws cicd, codepipeline
permalink: /blog/how-to-codepipeline-deploy-pipeline/
usemathjax: true
---

<figure>
    <img src="https://media.giphy.com/media/XIahGhbK5A685fyr8D/giphy.gif" class="img-fluid">
    <figcaption><small>어? 아니. 이게 왜? 아..!</small></figcaption>
</figure>

<!-- excerpt-start -->
프로젝트를 진행하며 우리 프로젝트에 배포 자동화를 도입해야 하는지 고민하는 경우가 많습니다. 배포할 때마다 혹여 실수로 서비스에 문제가 발생할까봐 걱정하고, 서버가 여러 대라면 배포 작업만으로도 피로를 느끼는 개발자들..

배포 자동화가 모든 문제를 해결해줄 수는 없지만, 배포에서 오는 피로감을 줄여주고, 배포 실수에 대한 책임을 개인에게서 떨쳐낼 수 있습니다. 배포 파이프라인을 구축하여 사용자가 실수하기 쉬운 따분하고 지루한 작업들을 자동화 해버립시다.

본 게시글은 AWS 환경에서 많이 사용되는 CodePipeline으로 배포 자동화를 구성하는 방법에 대해 설명합니다. 이 게시글을 읽고 배포 파이프라인을 구축하게 된다면, 분명 귀찮은 배포 과정으로부터 해방될 수 있을 것입니다


## 왜 배포를 자동화 해야할까?
모든 고민은 왜?(why)에서부터 시작됩니다. 배포 자동화가 필요한 하는 이유는 무엇일까요? 배포 자동화 방법은 다양하며, 이에 따른 이점도 다양합니다. 그 중 대표적인 이점들을 살펴보겠습니다.

### 빠른 서비스 릴리스

<figure>
    <img src="https://media.giphy.com/media/A0fBO6nktSpxk9rO4L/giphy.gif" class="img-fluid" width="200">
    <figcaption><small></small></figcaption>
</figure>

IT 기업, 특히 스타트업에서는 잦은 변경사항, 버그 수정으로 인해 수시로 배포가 이루어집니다. 이 때마다 개발자들은 본인이 작성한 소스코드를 서버들에 직접 들어가 배포하고, 서버를 재구동하며, 그 과정에서 서비스에 문제가 없는지를 점검합니다.

만약 서버가 여러 대 존재할 경우, 서비스의 중단을 방지하기 위해 다양한 배포 기법을 사용할 수도 있습니다(블루/그린, 롤링, 카나리, ...). 이 작업들을 모두 수동으로 수행한다면 배포 작업만으로도 상당 시간이 소요되며, 이는 서비스 운영에 버틀랙이 되어 버립니다.

보통 이런 배포의 과정은 정형화 되어 있으며, 배포할 때마다 다른 시나리오로 배포하는 경우는 거의 없습니다. 이 경우 일련의 과정을 스크립트화 하여 자동화하면, 개발자는 자신의 소스코드가 배포되는 과정을 그저 모니터링만 하면 되게 됩니다.

소스코드 작성부터 배포까지의 속도가 향상되니 **프로젝트 전반에 걸쳐 생산력이 증가하고, 개발자의 개발 부담도 덜게 되는 효과**를 기대할 수 있습니다.

### 배포 실수 줄이기

<figure>
    <img src="/assets/img/posts/2022-09-27/no-deploy-friday.jpeg" class="img-fluid" width="400">
    <figcaption><small></small></figcaption>
</figure>

배포는 매우 귀찮고 반복적인 작업이기 때문에 사용자 실수(human fault)를 일으키기 쉬우며, 배포 작업만으로도 개발자에게 상당한 피로감을 느끼게 해줍니다. 배포에 피로감을 느끼게 된다면, 다음 작업을 위한 추진력도 함께 꺾이게 됩니다.

>  A 대리: "B 씨, 금요일인데 퇴근안해?"<br>
>  B 신입: "개발하던게 거의 다 완료되어서요, 이것만 배포하고 갈게요!"<br>
> Q. 이 때, 'B 신입'이 집에 도착한 시간은?

위 증상은 IT 개발현장에서 쉽게 발견되는 증상으로, 금요일배포증후군이라고 불리는 매우 위중한 증상입니다 (No Deploy Fridays).
위와 같은 밈(meme)들이 많이 퍼져있는 것은, 그만큼 배포로 인한 다양한 문제를 겪어본 개발자들이 많다는 것을 의미합니다.

배포 자동화를 구성한다면, 이런 배포 문제들에 대한 방지가 가능해집니다. 문제의 원인이 소스코드에 있다면 **배포되기 전 테스트를 통해 잘못된 소스가 운영에 배포되는 것을 방지**하며, 배포 환경 설정이나 사용자의 실수에 의해 실패하는 경우는 **배포 과정을 스크립트화하여 실수를 사전에 방지**할 수 있습니다.


## 파이프라인 이란?

<figure>
    <img src="/assets/img/posts/2022-09-27/pipeline.jpg" class="img-fluid">
    <figcaption><small>"A씨, 여기 빌드 부분이 실패했어."</small></figcaption>
</figure>

그럼, 본격적으로 배포 자동화를 하기 위해서는 먼저 파이프라인이란 무엇인지에 대해 이해해야합니다. IT업계에서 사용되는 파이프라인이라는 용어는 **특정 단계의 결과물이 다음 작업의 입력(input)이 되는 구조를 의미합니다**.

이는 DevOps나 배포에서만 사용되는 용어가 아닌 컴퓨터 공학에 걸쳐 전반적으로 사용되는 용어입니다. 그렇다면 배포 파이프라인이란 무엇인가에 대해서도 자연스레 짐작해볼 수 있습니다.

하나의 서비스가 통합, 개발되어(CI), 테스트되고 배포되기(CD) 까지, 일련의 과정을 단계별로 정리하고 이를 순차적으로 실행하는 파이프라인을 일종의 배포 파이프라인이라 부릅니다.

> CI(Continuous Integration) : 직역하면 '지속적인 통합' 이란 뜻으로, 여러 개발자들이 함께 협업을 하며 서비스를 개발할 때, 이를 지속적으로 통합하고 중재하여 꾸준히 좋은 품질을 유지하려는 DevOps 관행 또는 그런 구조의 시스템을 의미합니다.

> CD(Continuous Deploy/Delivery) : CI가 완료된 프로그램을 운영 환경까지 매끄럽게 배포/전달 하도록 일련의 과정들을 자동화 하는 것을 의미합니다.

위와 같은 파이프라인의 정의를 이해하고 나면, 아래 설명드릴 파이프라인의 동작방식을 이해하기 쉬워집니다.


## 배포 파이프라인의 요소

<figure>
    <img src="/assets/img/posts/2022-09-27/conveyor.jpeg" class="img-fluid">
    <figcaption><small>"컨베이어 벨트와 닮은 파이프라인의 동작방식"</small></figcaption>
</figure>

배포 파이프라인은 각각 다른 역할을 수행하는 하위 단계들을 가지고 있습니다. 그 단계들은 순서대로 엮여 있어 **앞선 단계의 작업이 수행되어야 후행되는 작업이 시작하는 구조**를 가지고 있는데, 이런 모습을 컨베이어 벨트에 비유하기도 합니다.

- **Source**: 배포할 원본 소스코드를 뜻합니다. 빌드되지 않은 raw한 소스코드로 보통 git, svn의 원격 저장소를 지정합니다.
- **Build**: 소스코드를 빌드하는 단계입니다. maven, npm 과 같은 빌드 도구를 이용해 소스코드를 빌드 및 패키징합니다.
- **Deploy**: 빌드 완료된 프로그램을 실제 서버에 배포하는 단계입니다. 배포할 위치, 대상의 환경, 배포 방법 등을 선택할 수 있습니다.
- **etc..**: 수동 승인, 테스트, ... 등 위 세 가지 단계 이외에도 다양한 단계들을 추가하여, 파이프라인을 커스터마이징할 수 있습니다.

이러한 일련의 과정을 정리하고 코드화하여 파이프라인에 넘겨주게 되면, 파이프라인은 전달받은 프로세스를 하나씩 처리하며 매끄럽게 배포 작업을 수행해줄 것입니다.

위 단계들에 대해서는 아래에서 더 자세히 설명드리겠습니다. 위 단계들을 구축하기 위해서는 지금까지 개발자나 운영진이 자신의 소스코드를 운영의 서버에 배포하기까지 어떠한 절차들을 거쳐왔는지 점검하고 기록할 필요가 있습니다.

AWS는 이 각각의 단계에 대해 자체 관리형 서비스들을 제공하고 있으며, 서비스간의 연계성이 편리해 AWS 서비스만으로도 파이프라인을 구축하는 것이 가능합니다. (물론, 각 단계마다 반드시 AWS 서비스를 사용하실 필요는 없습니다)


### Source

Source 단계는 **CI 과정을 거쳐 배포될 소스코드를 선택하는 단계**입니다. 배포하고자 하는 소스코드가 담긴 레포지토리와 브런치를 선택해주면, 해당 브런치에 변경사항이 있을 때마다 CodePipeline을 트리깅하여 파이프라인을 동작시킵니다.

목적에 따라 서로 다른 브런치를 바라보는 것으로 개발계, 운영계 파이프라인을 나눌 수도 있으며, 하나의 파이프라인에 여러 레포지토리를 선택하여 각각의 레포지토리에서 원하는 데이터를 가져오게 할 수도 있습니다.

이처럼, Source 단계에서는 파이프라인, 서비스에 필요한 데이터가 저장된 저장소를 바라보는 단계입니다.
이 단계에서는 CodeCommit, Github, S3 와 같은 원격 저장소들을 선택할 수 있으며, S3를 제외하고는 모두 Git VCS(Version Control System)에 대한 원격 저장소들이 제공되고 있습니다.

> 현재 CodePipeline은 **Git**에 대한 원격저장소는 다양하게 지원하고 있으나, 그 외의 VCS에 대해서는 지원하지 않고 있습니다. 이 경우 별도의 추가 툴이 요구될 수 있으며, 오히려 **Jenkins**와 같은 다른 Pipeline 툴을 사용하는 것이 편리하실 수도 있습니다. (작성일: 2022-09-27)


### Build

Build 단계는 선택된 소스코드의 **빌드 또는 후처리 작업을 수행하는 단계**입니다. CodePipeline이 넘겨준 Source 단계의 소스코드를 지지고 볶는 단계로, raw한 소스코드를 빌드를 통해 컴퓨터가 수행 가능한 상태로 변환하거나, 또는 소스코드를 Deploy 하기에 앞서 패키징이나 후처리를 통해 결과물을 다듬어주는 역할을 수행합니다.

> Maven의 경우 mvn package, npm을 쓰는 경우 npm run build 를 하나의 예시로 들 수 있습니다.

Build 과정의 경우, 작업 특성상 소스코드에 대한 빌드를 직접 수행하는 단계이기 때문에, 개발자가 함께 참여하여 **빌드에 대한 커맨드를 작성해 Build 서버에 전달해주어야 합니다**. 이 단계에서는 CodeBuild, Jenkins를 사용하실 수 있으며, 필요하지 않은 경우 스킵하실 수 있습니다.

> 소스코드에 따라 빌드과정이 필요하지 않는 경우도 존재합니다. 이 경우는 CodePipeline을 생성하실 때, Build 단계를 건너뛰시고 바로 Deploy 단계를 구성하셔도 무방합니다.


### Deploy

Deploy 단계는 이전 과정까지 **완성된 결과물을 실제 서버에 배포해주는 역할을 수행합니다**.
Deploy 단계까지 도달하기 전에 소스코드는 이미 배포 가능한 완성품 상태여야하며, Deploy 단계는 이 결과물을 어떤 방식으로 어떤 서버의 어떤 위치에 배포할 것인지를 결정해야 합니다.

배포 대상과 방법은 매우 다양합니다(Blue Green, Rolling, 카나리, ...). 그 중 저희가 결과물을 어떤 서버에 배포하는지에 따라 선택할 수 있는 방법의 폭이 달라지게 되며, 몇몇 서비스들은 배포방법을 선택할 수 없는 경우도 존재합니다. (정해진 방식의 배포만 지원하는 경우)


## AWS CodePipeline

<figure>
    <img src="/assets/img/posts/2022-09-27/multipurpose_knife.jpeg" class="img-fluid">
    <figcaption><small>"AWS의 <del>단점</del> 장점."</small></figcaption>
</figure>

이제 본론으로 돌아와, AWS CodePipeline에 대해 알아보겠습니다. AWS는 수 많은 DevOps 서비스들을 제공하고 있습니다.
그 중 CodePipeline은 AWS Code 시리즈 중에서도 배포 파이프라인을 구축하는 역할을 수행합니다.

배포를 매끄럽게 수행할 수 있도록 컨베이어 벨트와 같은 역할을 수행해야 하는데, 이를 수행하기 위해 **CodePipeline은 각 단계별로 Event를 수신하는 방식을 사용합니다**. 시작은 소스단계이며, 소스단계에서 변경사항이 발생하면 이 Event를 캐치해 소스코드를 다음 단계로 넘겨줍니다.

그럼 Event를 전달받은 다음 단계가 작업을 수행하고 그 결과물을 CodePipeline에 전달합니다. 그러면 그 완료된 작업에 대한 Event를 수신한 CodePipeline은 이를 수신하여 또 다음 단계로 ... 이후부터는 여러분이 생각하시는 것과 동일합니다.

CodePipeline은 이렇듯 Event를 이용하여 각 단계를 연결시켜주고, 이에 대한 결과가 정상적인지 또는 실패했는지 확인합니다. 그러다 문제가 발견되면, 문제가 된 변경사항을 다음 단계로 넘기지 않도록 파이프라인을 중단시킵니다. 이렇듯 CodePipeline은 파이프라인에 대한 일종의 중재자 역할을 합니다.

> 이 때 각 단계에서 생성되어 다음 단계로 넘겨주는 데이터 또는 파일을 **아티팩트(artifact)**라고 부릅니다.


## 파이프라인 아키텍쳐 예시

<figure>
    <img src="/assets/img/posts/2022-09-27/cicd_1.png" class="img-fluid">
    <figcaption><small>"AWS 서비스로만 구성한 매우 심플한 배포 파이프라인 예시"</small></figcaption>
</figure>

위 아키텍처는 '배포 파이프라인의 요소' 목차에서 설명한 요소들로 구성한 예시 아키텍처입니다. 각 단계별로 AWS 서비스를 사용했으며, 전체 파이프라인에 대한 관리는 CodePipeline이 수행합니다.

위 아키텍처를 기반으로 예시 시나리오를 하나 들어보겠습니다.
1. 개발자가 소스코드를 작성한 후 `CodeCommit` 이라는 Git 원격 레포지토리에 변경사항을 push 합니다.
2. 소스의 변경사항을 주기적으로 모니터링하고 있던 `CodePipeline`은 해당 변경된 소스코드를 CodeCommit에서 가져옵니다.
3. CodePipeline은 가져온 소스코드를 다음 단계인 `CodeBuild`에 넘겨줘 빌드를 수행하도록 맡깁니다.
4. CodeBuild는 빌드를 수행하고, 빌드가 완료된 아티팩트를 CodePipeline에 다시 넘겨줍니다.
5. CodePipeline은 전달받은 아티팩트를 `S3` 버킷의 특정 위치에 넘겨줍니다.
6. S3는 전달받은 아티팩트를 이용하여 정적 웹 호스팅 기능을 통해 웹 서비스를 제공합니다.

위 시나리오에서 S3는 웹서버의 역할을 수행하기 때문에, S3가 아닌 EC2, ECS, ... 등 AWS에서 제공하는 여러 서버를 대입하여 생각하셔도 동일합니다. (다만 서비스별로 배포 구성방식은 다를 수 있습니다)


## 실제 구축해보기
지금까지 CodePipeline을 이용해 배포 파이프라인을 구축하는 방법과 예시에 대해 알아보았습니다. 이제 마지막 남은 단계는, 위 시나리오대로 실제로 구축을 진행해보는 것입니다.

위 **'파이프라인 아키텍쳐 예시'** 에서 설명드린 방식과 비슷하게 다양한 예제들을 구축해보시는 것을 추천드립니다. 소스코드를 저장할 레포지토리를 만들어 코드를 커밋하고, 빌드 서버를 추가하여 후처리 작업을 수행할 수 있도록 스크립트를 넘겨주며, 결과물을 서버에 배포하여 배포된 결과물을 확인합니다.

토이 프로젝트나 예제용 소스코드를 통해 파이프라인을 구축하여 결과물들을 확인하고, 해당 경험을 바탕으로 실제 서비스에서도 배포 파이프라인을 구축하게 된다면, 앞으로의 배포는 부담되는 작업이 아닌 결과물이 기대되는 작업이 될 것입니다.

아래 제가 작성한 다른 배포 예시들을 통해 다양한 예제를 경험해보실 것을 추천드립니다 :)

### 실습
- <small>S3를 활용한 실습: [CodePipeline으로 S3 배포 자동화하기][link_1]{:target="_blank"}</small>
- <small>ECS를 활용한 실습: CodePipeline으로 ECS 배포 자동화하기 (준비중..)

> AWS 콘솔은 주기적으로 많이 업데이트되고 변경됩니다. 단순히 화면을 따라하시기 보다는, 내용을 이해하시고 이에 부합한 버튼과 입력을 해주시면 되겠습니다.

## 출처

- <small>사진 출처(GIPHY) / <a href="https://media.giphy.com/media/XIahGhbK5A685fyr8D/giphy.gif" target="_blank">Computer Technology GIF - Sony Pictures Animation</a></small>
- <small>사진 출처(Freepik) / <a href="https://kr.freepik.com/free-vector/handymen-working-in-team-and-fixing-leakage-in-boiler-room-flat-vector-illustration-cartoon-plumbers-repairing-pipes-with-tools-flight-crew-and-aircraft-concept_11671671.htm#query=pipe&position=3&from_view=author" target="_blank">handymen-working-in-team-and-fixing... - pch.vector</a></small>
- <small>사진 출처(Freepik) / <a href="https://kr.freepik.com/free-vector/worker-watching-conveyor-with-boxes-isolated-flat-vector-illustration-cartoon-man-standing-in-warehouse-with-automation-process_10174056.htm#query=belt&position=44&from_view=author" target="_blank">worker-watching-conveyor-with-boxes - pch.vector</a></small>

[link_1]: https://creboring.net/blog/codepipeline-s3-deploy-automation/
