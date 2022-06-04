---
layout: post
title: "[CodePipeline] S3 CI/CD 파이프라인 구축하기 - Build"
summary: "소스코드를 빌드하여 배포본을 만듭니다."
author: creboring
date: '2022-05-16 22:26:47 +0530'
category: AWS
thumbnail: /assets/img/posts/category/CodePipeline.png
keywords: how to use code series, frontend aws cicd, frontend code series
permalink: /blog/codepipeline-s3-cicd-pipeline-3/
usemathjax: true
---

## 들어가며
---
이 글은 앞선 **[[CodePipeline] S3 CI/CD 파이프라인 구축하기 - Source 편][link_2]{:target="_blank"}** 을 잇는 **Build 편입니다**. 

이번 게시글에서는 Source 편에서 배포한 소스코드를 빌드하여 **배포할 수 있는 아티팩트를 만들 것입니다**. 예제 소스코드는 `Nodejs` 를 사용하고 있기 때문에 `npm` 패키지 매니저로 빌드를 수행합니다.
> AWS 콘솔은 주기적으로 많이 업데이트되고 변경됩니다. 단순히 화면을 따라하시기 보다는, 내용을 이해하시고 이에 부합한 버튼과 입력을 해주시면 되겠습니다 :)

## 목차
---
현재 게시글은 내용이 길어져 아래와 같이 단계별로 게시글이 나뉘어져 있습니다.
- 이론: [[CodePipeline] S3 CI/CD 파이프라인 구축하기 - 이론편][link_1]{:target="_blank"}
- 실습(Source): [[CodePipeline] S3 CI/CD 파이프라인 구축하기 - Source 편][link_2]{:target="_blank"}
- 실습(Build): [[CodePipeline] S3 CI/CD 파이프라인 구축하기 - Build 편][link_3]{:target="_blank"}
- 실습(Deploy): [[CodePipeline] S3 CI/CD 파이프라인 구축하기 - Deploy 편][link_4]{:target="_blank"}


## CodeBuild 란?
---
<figure>
    <img src="/assets/img/posts/2022-05-17/CodeBuild-icon.png" class="img-fluid" width="150px">
</figure>
`CodeBuild`는 AWS에서 제공하는 **관리형 빌드 서비스**입니다.

원하는 빌드 환경을 선택하고 스크립트를 제공하면, 이를 이용해 **소스코드를 빌드하며 결과물을 다음 스탭으로 전달해줍니다**.

이 과정에서 원하는 운영체제, 또는 VPC 환경 등을 선택하여 빌드 서버가 동작 할 네트워크 환경까지 세팅할 수 있습니다. (심지어는 OS나 라이브러리의 버전까지 선택할 수 있습니다.)

사용자가 빌드를 위해 해야할 일은 적절한 빌드 환경을 선택하고, 어떤 빌드 작업을 수행해야 하는지 스크립트를 전달하는 것 뿐입니다.


## CodeBuild 프로젝트 생성
---
CodeBuild는 Source 단계로부터 넘겨받은 소스코드를 통해 **배포 결과물**을 만들어내는 역할을 합니다. 

현 실습에선 CodeBuild에 Node.js 를 실행할 수 있는 환경을 세팅하고, **npm**, **webpack** 을 이용해 **패키징** 작업을 수행할 것입니다.

이 때, CodeBuild가 수행할 빌드 내용은 `buildspec.yml` 라는 스크립트 파일을 통해 제공할 것 입니다.

<img src="/assets/img/posts/2021-08-17/CodeBuild.png" class="img-fluid"/>

**<u># CodeBuild 프로젝트 생성</u>**
1. AWS 콘솔에서 **CodeBuild** 페이지로 이동합니다.
2. 프로젝트 생성 페이지로 넘어가서, 아래와 같이 항목들을 입력합니다.
   - 프로젝트 이름
   - 소스: AWS CodeCommit
      - 리포지토리: CodeCommit 레포지토리명
      - 브런치: master
   - 환경: 관리형 이미지
      - 운영 체제: Ubuntu
      - 런타임: Standard
      - 이미지: 최신 이미지 선택
      - 환경 유형: Linux
3. 이후 프로젝트 생성해줍니다.

<br>

### 빌드 스크립트 작성
---
아래는 이번 실습에 사용할 *buildspec.yml* 파일입니다. (소스코드 내에 포함되어 있습니다.)
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
위 스크립트를 이해하기에 앞서, CodeBuild는 일련의 **lifecycle**이 존재합니다. **install, pre_build, build, ... 등** 빌드의 각 단계를 도는데, 사용자는 이 lifecycle에 각각 원하는 스크립트를 넣어줄 수 있습니다. 

위 yml 파일의 경우 각각 다음과 같은 동작을 수행하도록 정의되어 있습니다.

**<u># Phases (단계) </u>**
- **install** : 빌드를 동작하기 위한 런타임 환경을 제공합니다. 이번 실습에서는 Nodejs, 그 중에서도 14 버전을 사용합니다.
- **pre_build** : npm을 통해 필요 모듈들 설치 (npm install)
- **build** : npm run script를 통해 패키징 작업 수행 (npm run build)

위 `npm run build`를 수행하고 나면 dist 라는 디렉토리에 결과물이 저장되는데, buildspec에서 **artifacts**의 **base-directory**로 `'dist'`, **files**로 `'**/*'`를 지정함으로써, dist 폴더 내의 모든 파일들을 다음 단계로 넘겨라는 의미의 스크립트가 작성됩니다.

> Code 시리즈에서는, 각 단계별로 서로 주고받는 데이터를 **artifact** 라고 부릅니다. 따라서, buildspec 파일의 artifacts 항목은, 다음 단계로 넘겨줄 아티팩트에 대한 정보를 담는 항목이며, **base-directory**는 넘겨줄 아티팩트의 base(root) 디렉토리, 그리고 **files**는 넘겨줄 파일을 뜻합니다. \*\*/\*는 모든 파일을 넘겨주겠다는 의미입니다.


## 정리
---
위 과정을 통해 **파이프라인에서 빌드 역할을 수행해줄 CodeBuild의 환경을 세팅했습니다**. buildspec.yml 파일의 경우, 이전에 업로드했던 Source 코드 내에 이미 포함되어 있기 때문에, 별도 수정할 내용 없이 바로 다음단계로 넘어가보겠습니다.

다음 단계는 마지막으로, 빌드 된 아티팩트를 어떻게 배포할지에 대해 다루며, 파이프라인을 구축해보겠습니다 :)
- 이전: [[CodePipeline] S3 CI/CD 파이프라인 구축하기 - Source 편][link_2]{:target="_blank"}
- 다음: [[CodePipeline] S3 CI/CD 파이프라인 구축하기 - Deploy 편][link_4]{:target="_blank"}


[link_1]: https://creboring.github.io/blog/codepipeline-s3-cicd-pipeline/
[link_2]: https://creboring.github.io/blog/codepipeline-s3-cicd-pipeline-2/
[link_3]: https://creboring.github.io/blog/codepipeline-s3-cicd-pipeline-3/
[link_4]: https://creboring.github.io/blog/codepipeline-s3-cicd-pipeline-4/