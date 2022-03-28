---
layout: post
title: "[CodePipeline] S3 CI/CD 파이프라인 구축하기 - Build"
summary: "CodePipeline을 이용한 S3 CI/CD 구축하기 - Build편"
author: creboring
date: '2022-03-09 17:47:30 +0530'
category: AWS
thumbnail: /assets/img/posts/category/CodePipeline.png
keywords: how to use code series, frontend aws cicd, frontend code series
permalink: /blog/codepipeline-s3-cicd-pipeline-3/
usemathjax: true
---


#### **3. CodeBuild 프로젝트 생성**
CodeBuild는 Source 단계로부터 넘겨받은 소스코드를 통해 **배포 결과물**을 만들어내는 역할을 합니다. 현 실습에선 CodeBuild를 통해 정적 소스코드에 npm, webpack으로 **패키징 작업**을 수행하고, 패키징의 결과물만을 배포 단계에 넘겨줄 것입니다. 이 때, CodeBuild는 `buildspec.yml` 파일을 통해 빌드 스크립트를 읽어옵니다.

<img src="/assets/img/posts/2021-08-17/CodeBuild.png" class="img-fluid"/>
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

아래는 현 게시글의 실습단계에서 사용할 *buildspec.yml* 파일입니다.
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
위 스크립트를 이해하기에 앞서, CodeBuild는 일련의 **lifecycle**이 존재합니다. **install, pre_build, build, ... 등**의 단계를 도는데, 사용자는 이 lifecycle에 원하는 스크립트를 넣어줄 수 있습니다. 위 yml 파일의 경우 각각 다음과 같은 동작을 수행하도록 정의되어 있습니다.
- **pre_build** : npm을 통해 필요 모듈들 설치 (npm install)
- **build** : npm run script를 통해 패키징 작업 수행 (npm run build)

위 `npm run build`를 수행하고 나면 dist 라는 디렉토리에 결과물이 저장되는데, buildspec의 **artifacts** **base-directory**로 `해당 dist 디렉토리`, **files**로 `'**/*'`를 지정함으로써, dist 폴더 내의 모든 파일들을 다음 단계로 넘겨라는 의미의 스크립트가 작성됩니다.

> Code 시리즈에서는, 각 단계별로 서로 주고받는 데이터의 단위를 **artifact** 로 정의합니다. 따라서, buildspec의 artifacts 항목은, 다음 단계로 넘겨줄 아티팩트에 대한 정보를 담는 항목이며, **base-directory**는 넘겨줄 아티팩트의 base(root) 디렉토리, 그리고 **files**는 넘겨줄 파일을 뜻합니다. \*\*/\*는 모든 파일을 넘겨주겠다는 의미입니다.
