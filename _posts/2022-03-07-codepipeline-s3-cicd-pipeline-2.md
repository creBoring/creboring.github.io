---
layout: post
title: "[CodePipeline] S3 CI/CD 파이프라인 구축하기 - Source 편"
summary: "CodePipeline을 이용한 S3 CI/CD 구축하기 - Source편"
author: creboring
date: '2022-03-07 22:37:18 +0530'
category: AWS
thumbnail: /assets/img/posts/category/CodePipeline.png
keywords: how to use code series, frontend aws cicd, frontend code series
permalink: /blog/codepipeline-s3-cicd-pipeline-2/
usemathjax: true
---


## **1. CodeCommit 레포지토리 생성**
---
**CodeCommit**은 AWS에서 제공하는 **Git 원격 레포지토리 서비스**로, Github와 같이 사용자의 Git 레포지토리를 원격 저장소에 저장할 수 있도록 해줍니다. Github 보다 부가 기능은 조금 부족하지만, 기본적인 Git 원격 레포지토리로써의 역할은 충실히 수행할 수 있으며, 무엇보다도 AWS의 **IAM 서비스를 통해 Git 동작에 대한 권한을 제어**할 수 있다는 큰 장점을 가지고 있습니다.

<img src="/assets/img/posts/2021-08-17/CodeCommit.png" class="img-fluid"/>
1. AWS 콘솔에 접속해 **CodeCommit** 페이지로 이동합니다.
2. 레포지토리 생성을 통해 '리포지토리 생성' 페이지로 이동한 후, 이름과 설명을 입력하고 레포지토리를 만들어줍니다.


## **2. AWS 권한 획득**
---
위 언급한 CodeCommit의 장점과 같이, CodeCommit은 기본적으로 **IAM 서비스**를 통한 접근제어가 가능합니다. 따라서, 저희는 CodeCommit에 샘플 소스를 업로드하기 위해 IAM 계정을 만들고 권한을 부여해주도록 하겠습니다. 그렇게 만들어진 권한은 Git 원격 레포지토리 로그인에 사용되며, 소스코드를 업로드할 권한을 얻게 됩니다.

<img src="/assets/img/posts/2021-08-17/IAM.png" class="img-fluid"/>

##### IAM 유저 생성
1. AWS 콘솔에서 **IAM** 페이지로 이동합니다.
2. **사용자 탭**에서 사용자 추가 페이지로 이동한 후, 이름과 액세스 유형으로 ***프로그래밍 방식 액세스***를 선택합니다.
3. 기존 정책 직접 설정으로 IAM 기본 정책을 직접 연결시켜 주겠습니다. (CodeCommit User에 부여하기 좋은 정책들을 이미 AWS에서 정의해두었습니다.) ***'AWSCodeCommitPowerUser'*** 정책을 찾아서 선택해줍니다.
4. 태그 추가는 넘어가도 되며, 사용자명을 입력하고 사용자를 생성해줍니다.

##### Git 자격증명 획득
1. IAM 사용자 탭에서 생성 된 사용자를 확인한 후, 사용자명을 클릭해 **요약 페이지**로 이동합니다.
2. 요약 페이지의 **보안 자격 증명** 탭으로 이동하면, Code Commit에 대한 **Git 자격증명**을 발급받을 수 있습니다. SSH와 HTTPS 두 종류를 받을 수 있는데, 현 실습에선 HTTPS 방식의 자격 증명을 생성해줍니다.
3. 화면에 표시된 사용자 정보는 저장해뒀다가 나중에 **Git 로그인**에 사용하시면 됩니다.

## **3. 샘플 소스코드 배포**
---
1. git이 설치되어 있지 않은 경우, [공식 홈페이지](https://git-scm.com/downloads) 로 이동하여 Git을 설치합니다.
2. 샘플 소스코드를 git을 통해 다운받습니다. ([샘플 소스코드](https://github.com/creBoring/S3-CICD-Sample.git)) (GitHub에 업로드 해두었습니다.)
3. 이후 CodeCommit의 레포지토리를 local로 가져옵니다. (CodeCommit 레포지토리의 URL은 AWS Console에서 확인하실 수 있습니다.)
```
ex) > git clone https://git-codecommit.(AWS 리전코드).amazonaws.com/v1/repos/(레포지토리)
```
이 때, git clone 과정에서 Git 로그인 정보를 물어보면, 아까 위에서 만들었던 ***IAM Git HTTPS 자격증명*** 을 입력해줍니다.
5. clone이 완료되면, 다운받은 샘플 소스코드를 CodeCommit 레포지토리로 복사합니다.
> ※ .git 디렉토리는 복사하지 않습니다.
6. 복사가 완료되었다면, **commit** 후 **push** 까지 완료해줍니다.
```
ex)
> git add .
> git commit -m "test"
> git push origin
```