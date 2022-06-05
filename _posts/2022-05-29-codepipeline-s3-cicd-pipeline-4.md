---
layout: post
title: "[CodePipeline] S3 CI/CD 파이프라인 구축하기 - Deploy"
summary: "전 과정을 연결하는 CodePipeline을 구축합니다."
author: creboring
date: '2022-05-28 20:52:12 +0530'
category: AWS
thumbnail: /assets/img/posts/category/CodePipeline.png
keywords: how to use code series, frontend aws cicd, frontend code series
permalink: /blog/codepipeline-s3-cicd-pipeline-4/
usemathjax: true
---

이 글은 앞선 **[[CodePipeline] S3 CI/CD 파이프라인 구축하기 - Build 편][link_2]{:target="_blank"}** 을 잇는 **Deploy 편입니다**.

이번 게시글에서는 Build 편에서 패키징된 소스코드를 `Amazon S3`에 배치하는 **CodePipeline 파이프라인**을 만들 것입니다.
> AWS 콘솔은 주기적으로 많이 업데이트되고 변경됩니다. 단순히 화면을 따라하시기 보다는, 내용을 이해하시고 이에 부합한 버튼과 입력을 해주시면 되겠습니다 :)

## 목차
---
현재 게시글은 내용이 길어져 아래와 같이 단계별로 게시글이 나뉘어져 있습니다.
- 이론: [[CodePipeline] S3 CI/CD 파이프라인 구축하기 - 이론편][link_1]{:target="_blank"}
- 실습(Source): [[CodePipeline] S3 CI/CD 파이프라인 구축하기 - Source 편][link_2]{:target="_blank"}
- 실습(Build): [[CodePipeline] S3 CI/CD 파이프라인 구축하기 - Build 편][link_3]{:target="_blank"}
- 실습(Deploy): [[CodePipeline] S3 CI/CD 파이프라인 구축하기 - Deploy 편][link_4]{:target="_blank"}


## Amazon S3 란?
--- 
<figure>
    <img src="/assets/img/posts/2022-05-29/AmazonS3-icon.png" class="img-fluid" width="150px">
</figure>
`Amazon S3`란 **AWS에서 제공해주고 있는 스토리지 서비스**입니다. 다만, S3는 일반적으로 생각하는 파일 스토리지가 아닌 **객체형 스토리지 서비스**입니다. 

간단하게 생각하자면, PC에 사용되는 **파일 스토리지**는 폴더라는 개념을 통해 파일들을 계층적으로 관리하지만, **객체형 스토리지**는 이런 계층 구조 없이 평면 구조로 실제 데이터를 저장하고, 이에 대한 메타데이터를 검색하여 접근하는 방식을 사용합니다.

각각의 스토리지 구조마다 장단점이 있지만, 클라우드라는 개념 위에서 객체형 스토리지 서비스는 무한한 확장성과, 방대한 데이터에 대한 검색이 빠르다는 장점이 매우 잘 활용됩니다.

Amazon S3는 여기에 추가적으로, **'Static Web Hosting'** 이라는 기능을 제공하는데, 객체형 스토리지에 저장된 데이터들을 가지고 웹 서버를 호스팅해주는 기능을 제공합니다.

매우 쉽고 간단하게 웹 서버를 호스팅할 수 있기 때문에, 단순한 정적 페이지에 대한 호스팅이 필요하다면 편리하게 사용할 수 있는 기능입니다.


## S3 버킷 설정
---
위에서 언급한 사항과 같이, S3는 기본적으로 데이터 저장소로써 많이 사용되지만 Web 서버의 역할도 수행할 수 있습니다. 

이번에는 소스코드를 배포할 **S3 버킷**을 새로 생성하고, ***Static Web Hosting 기능*** 을 활성화 하여 Web 서버를 띄워보겠습니다.
> ※ 실제 서비스 도메인을 S3 정적 웹 호스팅 도메인에 연결시킬 경우, S3 버킷명은 서비스 도메인명과 일치해야 합니다. <br>
ex) 버킷명 = test.creboring.com

<img src="/assets/img/posts/2021-08-17/S3.png" class="img-fluid"/>

<br>

### S3 버킷 생성
---
버킷 생성은 매우 간단합니다. 아래 절차를 따라 S3 서비스 페이지에서 버킷을 하나 생성합니다.

1. AWS 콘솔에서 **S3** 페이지로 이동합니다.
2. 버킷 만들기 페이지로 이동해 버킷명을 입력하고 생성합니다. (버킷명은 서비스 도메인명과 일치해야 합니다.)

<br>

### S3 버킷 권한 설정
---
생성된 S3 버킷은 **기본적으로 외부에서 접근이 불가능합니다**. 저희는 해당 버킷으로 웹 서비스를 할 것이기 때문에, 아래 절차를 통해 권한을 모두 열어줍시다.

1. 방금 생성한 S3 버킷의 상세 페이지로 이동합니다.
2. 페이지의 **권한** 탭에서 **퍼블릭 액세스 차단**을 모두 비활성화해줍니다. (버킷에 외부 사용자가 접근 가능하게 하기 위한 용도입니다)
3. 같은 **권한** 탭 내에서 **버킷 정책**을 설정해줍니다. 버킷 정책으로 아래 json을 입력해줍니다. (버킷 ARN에는 실제 버킷의 ARN을 입력해줍니다) (마찬가지로 버킷 객체에 외부 사용자가 접근 가능하도록 하기 위함입니다.)
``` json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "(버킷 ARN)"
    }
  ]
}
```

<br>

### S3 정적 웹 호스팅 활성화
---
이제 S3 버킷에 정적 웹 호스팅 기능을 활성화합니다. 

1. 버킷 상세 페이지에서 **속성** 탭으로 이동합니다.
2. 속성 탭에서 정적 웹 사이트 호스팅을 활성화 해줍니다. 활성화 시 아래와 같은 설정을 선택해줍니다.
    - 호스팅 유형: 정적 웹 사이트 호스팅
    - 인덱스 문서: index.html


## CodePipeline 생성
---
이제 각 단계를 파이프라인으로 연결시켜, CodeCommit에 커밋이 발생했을 때 자동으로 빌드와 배포가 수행되도록 구성합니다. **CodePipeline**은 이 **모든 단계들을 연결시켜주는 매개체**이며, source, build, deploy 외에도 *test* , *수동 승인* 과 같은 추가 작업들을 파이프라인에 추가할 수 있습니다.

<img src="/assets/img/posts/2021-08-17/CodePipeline.png" class="img-fluid"/>
1. AWS 콘솔에서 **CodePipeline** 페이지로 이동합니다.
2. 파이프라인 생성 페이지로 이동한 후, 한 단계씩 지금까지 생성한 리소스들을 추가해주겠습니다.
3. **소스(Source)** 단계에서는 아래와 같이 선택합니다.
   - **소스 공급자**: AWS CodeCommit
   - **리포지토리 이름**: (위에서 만든 CodeCommit 레포지토리)
   - **브랜치 이름**: master
4. **빌드(Build)** 단계에서는 아래와 같이 선택합니다.
   - **빌드 공급자**: AWS CodeBuild
   - **리전**: 아시아 태평양(서울)
   - **프로젝트 이름**: (위에서 만든 CodeBuild 프로젝트)
5. **배포(Deploy)** 단계에서는 아래와 같이 선택합니다.
   - **배포 공급자**: Amazon S3
   - **리전**: 아시아 태평양(서울)
   - **버킷**: (정적 웹호스팅 기능이 활성화 된 S3 버킷)
   - **'배포하기 전에 파일 압축 풀기'** 를 활성화해줍니다.
6. 마지막으로 입력한 내용들 검토 후 파이프라인을 생성합니다.

> ※ CodePipeline이 생성되는 과정에서, IAM 관련 리소스가 즉시 업데이트 되지 않아 첫 배포가 실패될 수 있습니다. 이 경우 **[변경 사항 릴리즈]** 버튼이나 **[재시도]** 버튼을 통해 배포를 재시도해보시기 바랍니다.

## 테스트
---
파이프라인이 모두 정상적으로 구축될 경우, 아래 사진과 같이 모든 단계가 성공으로 표시되어야 합니다. 만약 그렇지 않은 경우 상세 보기를 통해 실패의 원인을 확인해보시기 바랍니다.

<img src="/assets/img/posts/2021-08-17/CodePipeline_Success.png" class="img-fluid"/>

파이프라인이 성공적으로 구동한 경우, S3 버킷에도 아래와 같이 정상적으로 소스들이 확인되셔야 합니다.

<img src="/assets/img/posts/2021-08-17/S3_Success.png" class="img-fluid"/>

이제 S3 버킷 상세페이지의 속성 탭으로 이동하여 정적 웹호스팅 URL을 확인하고, 외부에서 정상적으로 호출이 가능한지 테스트해봅니다.

<img src="/assets/img/posts/2021-08-17/S3_Web.png" class="img-fluid"/>

URL 호출 시, 아래와 같이 확인되시면 성공입니다.

<img src="/assets/img/posts/2021-08-17/Success.png" class="img-fluid"/>

## 끝으로
---
지금까지 CodePipeline을 이용해 **S3 버킷을 향한 CI/CD Pipeline을 구축해보았습니다**. 짧지만 어렵게 느껴질 수 있을만한 내용으로, AWS와 CI/CD에 대한 이해도에 따라 다르게 느껴지실 것 같습니다.

이번 실습에 사용된 예제나 파이프라인은 매우 단순한 방식의 예제입니다만, 실제로는 운영환경이나 실용성을 위해 추가적인 단계가 많이 사용되기도 하며, Lambda Slack Bot 등을 이용해 수동 승인 절차를 추가하기도 합니다.

현 게시글에서 이를 모두 다루지는 못했지만, 가장 간단한 방식의 파이프라인을 따라 구축해보며 파이프라인이 어떻게 만들어지고 구성되는지에 대해 조금이나마 이해하셨다면 성공적이라고 생각합니다. 현재 게시글에 대해 추가적인 질문이나, 잘못된 내용이 있다면 댓글로 남겨주시면 감사하겠습니다

감사합니다. 수고하셨습니다.

[link_1]: https://creboring.github.io/blog/codepipeline-s3-cicd-pipeline/
[link_2]: https://creboring.github.io/blog/codepipeline-s3-cicd-pipeline-2/
[link_3]: https://creboring.github.io/blog/codepipeline-s3-cicd-pipeline-3/
[link_4]: https://creboring.github.io/blog/codepipeline-s3-cicd-pipeline-4/