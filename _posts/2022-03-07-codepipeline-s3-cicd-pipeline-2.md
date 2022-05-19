---
layout: post
title: "[CodePipeline] S3 CI/CD 파이프라인 구축하기 - Source 편"
summary: "CodeCommit에 소스코드를 배포하여 파이프라인의 소스를 구성합니다."
author: creboring
date: '2022-03-07 22:37:18 +0530'
category: AWS
thumbnail: /assets/img/posts/category/CodePipeline.png
keywords: how to use code series, frontend aws cicd, frontend code series
permalink: /blog/codepipeline-s3-cicd-pipeline-2/
usemathjax: true
---

## **들어가며**
---
이 글은 앞선 **[[CodePipeline] S3 CI/CD 파이프라인 구축하기 - 이론편][link_1]{:target="_blank"}** 을 잇는 **Source 편입니다**. 

이번 게시글에서는 이론편에서 설명한 **S3 CI/CD 파이프라인의 소스 단계를 구축합니다**. 형상관리 툴로는 `Git`을 사용할 예정이며, AWS 관리형 서비스인 `CodeCommit` 을 사용하여 원격 레포지토리 생성 후 소스코드를 배포해보겠습니다.
> AWS 콘솔은 주기적으로 많이 업데이트되고 변경됩니다. 단순히 화면을 따라하시기 보다는, 내용을 이해하시고 이에 부합한 버튼과 입력을 해주시면 되겠습니다 :)

## 목차
---
현재 게시글은 내용이 길어져 아래와 같이 단계별로 게시글이 나뉘어져 있습니다.
- 이론: [[CodePipeline] S3 CI/CD 파이프라인 구축하기 - 이론편][link_1]{:target="_blank"}
- 실습(Source): [[CodePipeline] S3 CI/CD 파이프라인 구축하기 - Source 편][link_2]{:target="_blank"}
- 실습(Build): [[CodePipeline] S3 CI/CD 파이프라인 구축하기 - Build 편][link_3]{:target="_blank"}
- 실습(Deploy): [[CodePipeline] S3 CI/CD 파이프라인 구축하기 - Deploy 편][link_4]{:target="_blank"}


## CodeCommit 이란?
---
<figure>
    <img src="/assets/img/posts/2022-03-07/CodeCommit-icon.png" class="img-fluid" width="200px">
</figure>
`CodeCommit`은 AWS에서 제공하는 **Git 원격 레포지토리 서비스**로, Github와 같이 사용자의 Git 레포지토리를 원격 저장소에 저장할 수 있도록 해줍니다. 

Github 보다 부가 기능은 조금 부족하지만, 기본적인 Git 원격 레포지토리로써의 역할은 충실히 수행할 수 있으며, AWS의 **IAM 서비스를 통해 Git 동작에 대한 권한을 제어**할 수 있다는 장점도 가지고 있습니다.

그 대신, CodeCommit 을 사용하기 위해서는 **Git 사용자**를 만들어 주기 위해 필수적으로 **IAM User**를 만들어주어야 합니다. (모든 권한 관리는 IAM 으로부터...)


## CodeCommit 레포지토리 생성
---
<img src="/assets/img/posts/2022-03-07/CodeCommit.png" class="img-fluid"/>

`CodeCommit` 레포지토리는 클릭 몇 번으로 간단하게 생성이 가능합니다. 아래 절차를 따라 AWS Console 웹 화면에서 레포지토리 이름을 입력해주고 원격 레포지토리를 만들어줍시다.

1. AWS 콘솔에 접속해 **CodeCommit** 페이지로 이동합니다.
2. 레포지토리 생성을 통해 **'리포지토리 생성'** 페이지로 이동한 후, 이름과 설명을 입력하고 레포지토리를 만들어줍니다.


## AWS 권한 획득
---
위 언급한 내용과 같이, CodeCommit은 기본적으로 `IAM` 서비스를 통해 접근제어가 가능합니다. 

**CodeCommit 원격 레포지토리에 접근하기 위한 Git 사용자도 IAM을 통해 생성 및 관리되는데**, 저희는 CodeCommit에 샘플 소스를 업로드하기 위해 IAM 사용자를 만들고 권한을 부여해주도록 하겠습니다.
> IAM(Identity and Access Management): AWS 리소스들에 대한 접근 권한, 사용자 등을 관리해주는 AWS 대표 보안 서비스입니다. 사용자(User)를 만들고 사용자별로 권한을 달리 부여할 수 있으며, 이는 CodeCommit에 대한 권한도 마찬가지입니다.

<img src="/assets/img/posts/2022-03-07/IAM.png" class="img-fluid"/>

##### **<u># IAM 유저 생성</u>**
1. AWS 콘솔에서 **IAM** 페이지로 이동합니다.
2. **사용자 탭**에서 사용자 추가 페이지로 이동한 후, 이름과 액세스 유형으로 ***프로그래밍 방식 액세스***를 선택합니다.
3. 기존 정책 직접 설정으로 IAM 기본 정책을 직접 연결시켜 주겠습니다. (CodeCommit User에 부여하기 좋은 정책들을 이미 AWS에서 정의해두었습니다.) ***'AWSCodeCommitPowerUser'*** 정책을 찾아서 선택해줍니다.
4. 태그 추가는 넘어가도 되며, 사용자명을 입력하고 사용자를 생성해줍니다.

<br>

##### **<u># Git 사용자 Credential 획득</u>**
1. IAM 사용자 탭에서 생성 된 사용자를 확인한 후, 사용자명을 클릭해 **요약 페이지**로 이동합니다.
2. 요약 페이지의 **보안 자격 증명** 탭으로 이동하면, Code Commit에 대한 **Git 자격증명**을 발급받을 수 있습니다. SSH와 HTTPS 두 종류를 받을 수 있는데, 현 실습에선 HTTPS 방식의 자격 증명을 생성해줍니다.
3. 화면에 표시된 사용자 정보는 저장해뒀다가 나중에 **Git 로그인**에 사용하시면 됩니다.

## 샘플 소스코드 배포
---
1. git이 설치되어 있지 않은 경우, [공식 홈페이지][link_5]{:target="_blank"} 로 이동하여 `Git`을 설치합니다.
2. 샘플 소스코드를 git을 통해 다운받습니다. ([샘플 소스코드][link_6]{:target="_blank"}) (GitHub에 업로드 해두었습니다.)
3. 이후 CodeCommit의 레포지토리를 local로 가져옵니다. (CodeCommit 레포지토리의 URL은 AWS Console에서 확인하실 수 있습니다.)
```
ex) > git clone https://git-codecommit.(AWS 리전코드).amazonaws.com/v1/repos/(레포지토리)
```
이 때, git clone 과정에서 Git 로그인 정보를 물어보면, 아까 위에서 만들었던 **IAM Git HTTPS 자격증명** 을 입력해줍니다.
5. clone이 완료되면, 다운받은 샘플 소스코드를 CodeCommit 레포지토리로 복사합니다.
> ※ .git 디렉토리는 복사하지 않습니다.
6. 복사가 완료되었다면, **commit** 후 **push** 까지 완료해줍니다.
```
ex)
> git add .
> git commit -m "test"
> git push origin
```


## 확인
---
소스코드가 원격 레포지토리에 정상적으로 업로드 되었는지 확인합니다. 아래와 같이 소스코드들이 업로드 되어 있으면 정상입니다.

<img src="/assets/img/posts/2022-03-07/CodeCommit-done.png" class="img-fluid"/>


## 정리
---
위 과정을 통해 **파이프라인을 동작시킬 소스코드 저장소를 만들고, 소스코드를 업로드했습니다**. 다음 과정에서는 업로드된 소스코드를 어떻게 빌드할지에 대해 다룹니다.
- 이전: [[CodePipeline] S3 CI/CD 파이프라인 구축하기 - 이론편][link_1]{:target="_blank"}
- 다음: [[CodePipeline] S3 CI/CD 파이프라인 구축하기 - Build 편][link_3]{:target="_blank"}

[link_1]: https://creboring.github.io/blog/codepipeline-s3-cicd-pipeline/
[link_2]: https://creboring.github.io/blog/codepipeline-s3-cicd-pipeline-2/
[link_3]: https://creboring.github.io/blog/codepipeline-s3-cicd-pipeline-3/
[link_4]: https://creboring.github.io/blog/codepipeline-s3-cicd-pipeline-4/
[link_5]: https://git-scm.com/downloads
[link_6]: https://github.com/creBoring/S3-CICD-Sample.git