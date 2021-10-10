---
layout: post
title: "[CodePipeline] S3 CI/CD 파이프라인 구축하기"
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
본 게시글은 주로 프런트엔드(Web)에서 많이 사용되는 정적 소스코드를 ***Amazon S3*** 에 **자동**으로 **배포**하는 과정을 다룹니다. CI/CD 구축은 프런트엔드, 백엔드 구분 없이 소스코드에 대한 빌드와 배포과정을 자동화하여 개발자로 하여금 개발에만 집중할 수 있는 환경을 제공해줍니다.

해당 자동화 방법이나 배포 대상은 매우 다양하나, 현 게시글에서는 ***Amazon Web Service*** 의 **Code 시리즈**를 사용하여 **S3 정적 웹 호스팅**에 배포하는 방법을 다루고 있습니다. Code 시리즈, 또는 S3에 대해 생소하신 분들은 아래 게시글을 먼저 읽고 오실 것을 권고드립니다.
> AWS Code 시리즈란?
[게시글 읽기][link_1]{:target="_blank"}

> AWS S3 정적 웹 호스팅이란?
[게시글 읽기][link_2]{:target="_blank"}


<br>
## ✔️ 아키텍쳐
---
<img src="/assets/img/posts/2021-08-17/cicd_1.png" class="img-fluid">
- Pipeline : **CodePipeline**<br>
- Source : **CodeCommit**<br>
- Build : **CodeBuild**<br>
- Deploy : **S3**

<br>
정적 컨텐츠의 경우 **빌드 과정이 필수가 되지는 않습니다.** npm이나 별도 패키징 및 플러그인을 사용하지 않는 경우 빌드 과정을 생략하는 경우도 있으며, 빌드를 동작시키지 않아도 배포가 가능한 경우도 있습니다.

다만, 대부분의 소스코드들이 raw한 소스코드를 그대로 운영서버에서 사용하지 않기 때문에, 소스코드에 대한 **패키징 작업**이나 **후처리 작업**을 빌드 과정에서 수행해줄 수 있습니다.


<br>
## ⚙️ 구축 방법
---
위에서도 언급했듯이 CI/CD는 개발자로 하여금 개발에만 집중할 수 있도록 도와주는 역할을 합니다. 실제 구축에 앞서, 아래 각 파이프라인 단계의 구축이 어떤 방향으로 진행되는지 알아보겠습니다. 먼저, CodePipeline은 아래와 같은 단계가 존재합니다.
- Source
- Build
- Deploy
- etc.. (수동 승인, 테스트, ...)

이 중에서도 단연 중요한 단계는 **Source, Build, Deploy** 단계인데, 각 단계는 서로 **분리된 목적**으로 각각의 역할을 수행하며, AWS는 이 각각의 단계에 대해 자체 관리형 서비스들을 제공하고 있습니다. (물론, 각 단계마다 반드시 AWS 서비스를 사용하실 필요는 없습니다) 그럼, 각 단계에서 수행하는 역할과, AWS에서 제공하는 서비스에 대해 알아보겠습니다.

<br>
#### **Source**
Source 단계는 **CI 과정을 거쳐 배포될 소스코드를 선택하는 단계**입니다. 배포하고자 하는 소스코드가 담긴 **레포지토리**와 **브런치**를 선택해주면, 해당 브런치에 변경사항이 있을 때마다 CodePipeline을 트리깅하여 파이프라인을 동작시킵니다.

목적에 따라 서로 다른 브런치를 바라보는 것으로 **개발계**, **운영계** 파이프라인을 나눌 수도 있으며, **하나의 파이프라인에 여러 레포지토리**를 선택하여 각각의 레포지토리에서 원하는 데이터를 가져오게 할 수도 있습니다.

이처럼, Source 단계에서는 파이프라인, 서비스에 필요한 데이터가 저장된 **저장소**를 바라보는 단계입니다.<br>
이 단계에서는 CodeCommit, Github, S3 와 같은 원격 저장소들을 선택할 수 있으며, S3를 제외하고는 모두 **Git VCS(Version Control System)**에 대한 원격 저장소들이 제공되고 있습니다.

> *※ 현재 CodePipeline은 Git에 대한 원격저장소는 다양하게 지원하고 있으나, 그 외의 VCS에 대해서는 지원하지 않고 있습니다. 이 경우 별도의 추가 툴이 요구될 수 있으며, 오히려 Jenkins와 같은 다른 Pipeline 툴을 사용하시는 것이 편리하실 수도 있습니다. (작성일: 2021-09-24)*

<br>
#### **Build**
Build 단계는 선택된 소스코드의 **빌드**, 또는 **후처리 작업**을 수행하는 단계입니다. CodePipeline이 넘겨준 Source 단계의 소스코드를 지지고 볶는 단계로, raw한 소스코드를 빌드를 통해 **컴퓨터가 수행 가능한 상태로 변환**하거나, 또는 소스코드를 Deploy 하기에 앞서 **패키징**이나 **후처리**를 통해 결과물을 다듬어주는 역할을 수행합니다.

Maven의 경우 `mvn package`, npm을 쓰는 경우 `npm install`, `npm run build` 를 하나의 예시로 들 수 있습니다.

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
Deploy 단계는 이전 과정까지 완성된 **결과물**을 **실제 서버에 배포**해주는 역할을 수행합니다. Deploy 단계까지 도달하기 전에 소스코드는 이미 **배포 가능한 완성품 상태**여야하며, Deploy 단계는 이 결과물을 **어떤 방식**으로 **어떤 서버**의 **어떤 위치**에 배포할 것인지를 결정해야 합니다.

배포 대상과 방법은 매우 다양합니다(Blue Green, Rolling, ...). 그 중 저희가 결과물을 어떤 서버에 배포하는지에 따라 선택할 수 있는 방법의 폭이 달라지게 되며, 몇몇 서비스들은 배포방법을 지정하지 않아도 대상 서비스에서 자체적으로 배포를 수행합니다.



<br>
## 💻 실습
---
실습은 위 **'구축 방법'** 에서 설명드린 순서대로 진행됩니다.

먼저, 소스코드를 저장할 ***레포지토리*** 를 만들어 코드를 커밋하고, ***빌드 서버*** 를 추가하여 후처리 작업을 수행할 수 있도록 ***스크립트*** 를 넘겨줄 예정입니다. 마지막으로 결과물을 ***S3*** 에 배포하도록 파이프라인에 지정하면 끝입니다. 이 모든 과정은 **CodePipeine**에 의해 트리깅되며, CodePipeline은 그 실행 결과를 사용자에게 보여줍니다.

이제 실제로 AWS Console에 접속하여, 실습을 진행해보겠습니다.
> AWS 콘솔은 주기적으로 많이 업데이트되고 변경됩니다. 단순히 화면을 따라하시기 보다는, 내용을 이해하시고 이에 부합한 버튼과 입력을 해주시면 되겠습니다 :)

#### **목차**
1. CodeCommit 레포지토리 생성
2. 샘플 소스코드 배포
3. S3 버킷 생성
4. CodeBuild 프로젝트 생성
5. CodePipeline 생성
6. 테스트

<br>
#### **1. CodeCommit 레포지토리 생성**
**CodeCommit**은 AWS에서 제공하는 **Git 원격 레포지토리 서비스**로, Github와 같이 사용자의 Git 레포지토리를 원격 저장소에 저장할 수 있도록 해줍니다. Github 보다 부가 기능은 조금 부족하지만, 기본적인 Git 원격 레포지토리로써의 역할은 충실히 수행할 수 있으며, 무엇보다도 AWS의 **IAM 서비스를 통해 Git 동작에 대한 권한을 제어**할 수 있다는 큰 장점을 가지고 있습니다.

<img src="/assets/img/posts/2021-08-17/CodeCommit.png" class="img-fluid"/>
1. AWS 콘솔에 접속해 **CodeCommit** 페이지로 이동합니다.
2. 레포지토리 생성을 통해 '리포지토리 생성' 페이지로 이동한 후, 이름과 설명을 입력하고 레포지토리를 만들어줍니다.

<br>
#### **2. 샘플 소스코드 배포**
위 언급한 CodeCommit의 장점과 같이, CodeCommit은 기본적으로 **IAM 서비스**를 통한 접근제어가 가능합니다. 따라서, 저희는 CodeCommit에 업로드하기 위해 IAM 계정을 만들고 권한을 부여해주도록 하겠습니다. 그렇게 만들어진 권한은 Git 원격 레포지토리 로그인에 사용되며, 소스코드를 업로드할 권한을 얻게 됩니다.


<img src="/assets/img/posts/2021-08-17/IAM.png" class="img-fluid"/>
##### IAM 유저 생성
1. AWS 콘솔에서 **IAM** 페이지로 이동합니다.
2. **사용자 탭**에서 사용자 추가 페이지로 이동한 후, 이름과 액세스 유형으로 ***프로그래밍 방식 액세스***를 선택합니다.
3. 기존 정책 직접 설정으로 IAM 기본 Policy를 직접 연결시켜 주겠습니다. (CodeCommit User에 부여하기 좋은 정책이 이미 정의되어 있습니다.) ***'AWSCodeCommitPowerUser'*** 권한을 찾아서 선택해줍니다.
4. 태그 추가는 넘어가도 되며, 사용자명을 입력하고 사용자를 생성해줍니다.

##### Git 자격증명 획득
1. IAM 사용자 탭에서 생성 된 사용자를 확인한 후, 사용자명을 클릭해 **요약 페이지**로 이동합니다.
2. 요약 페이지의 **보안 자격 증명** 탭으로 이동하면, Code Commit에 대한 **Git 자격증명**을 발급받을 수 있습니다. SSH와 HTTPS 두 종류를 받을 수 있는데, 현 실습에선 HTTPS 방식의 자격 증명을 생성해줍니다.
3. 화면에 표시된 사용자 정보는 저장해뒀다가 나중에 **Git 로그인**에 사용하시면 됩니다.

##### 샘플 소스코드 배포
1. git이 설치되어 있지 않은 경우, [공식 홈페이지](https://git-scm.com/downloads) 로 이동하여 Git을 설치합니다.
2. 샘플 소스코드를 git을 통해 다운받습니다. ([샘플 소스코드](https://github.com/creBoring/S3-CICD-Sample.git)) (GitHub에 업로드 해두었습니다.)
3. 이후 CodeCommit의 레포지토리를 local로 가져옵니다. (CodeCommit 레포지토리의 URL은 AWS Console에서 확인하실 수 있습니다.)
```
ex) > git clone https://git-codecommit.(AWS 리전코드).amazonaws.com/v1/repos/(레포지토리)
```
이 때, git clone 과정에서 Git 로그인 정보를 물어보면, 아까 위에서 만들었던 ***IAM Git HTTPS 자격증명*** 을 입력해줍니다.
5. clone이 완료되면, 아까 다운받은 샘플 소스코드를 CodeCommit 레포지토리로 복사합니다.
> ※ .git 디렉토리는 복사하지 않습니다.
6. 복사가 완료되었다면, **commit** 후 **push** 까지 완료해줍니다.
```
ex)
> git add .
> git commit -m "test"
> git push origin
```

<br>
#### **3. S3 버킷 생성**

<br>
#### **4. CodeBuild 프로젝트 생성**

<br>
#### **5. CodePipeline 생성**

<br>
#### **6. 테스트**

<br>
## ☑️ 끝으로
---
힘덜다

[link_1]: https://creboring.github.io/blog/what-is-code-series/
[link_2]: https://creboring.github.io/blog/what-is-s3-static-web-hosting/
