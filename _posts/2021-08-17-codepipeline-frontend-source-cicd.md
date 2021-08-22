---
layout: post
title: "AWS CodePipeline을 이용해 Front-end 자동 배포하기"
summary: "CodePipeline을 이용한 Front-end 소스코드 CI/CD"
author: creboring
date: '2021-08-17 9:52:20 +0530'
category: AWS
thumbnail: /assets/img/posts/CodePipeline.png
keywords: how to use code series, frontend aws cicd, frontend code series
permalink: /blog/codepipeline-frontend-source-cicd/
usemathjax: true
---
## 개요
---
본 게시글은 프런트엔드 소스코드를 ***S3 정적 웹 호스팅***에 **자동**으로 **배포**하는 과정을 다룹니다.

프런트엔드, 백엔드 구분 없이 소스코드에 대한 CI/CD 구축은 빌드와 배포과정을 자동화하여 개발자로 하여금 개발에만 집중할 수 있도록 해줍니다.

해당 자동화 방법이나 배포 대상은 매우 다양하나, 현 게시글에서는 ***Amazon Web Service*** 의 **Code 시리즈**를 사용하여 **S3 정적 웹 호스팅**에 배포하는 방법을 다루고 있습니다.

Code 시리즈, 또는 S3에 대해 생소하신 분들은 아래 게시글을 먼저 읽고 오실 것을 권고드립니다.
> AWS Code 시리즈란?<br>
https://creboring.github.io/blog/what-is-code-series/

> AWS S3 정적 웹 호스팅이란?<br>
https://creboring.github.io/blog/what-is-s3-static-web-hosting/

<br>
## 아키텍쳐
---
<img src="/assets/img/posts/2021-08-17-codepipeline-frontend-source-cicd_1.png" class="img-fluid">
- Pipeline : **CodePipeline**<br>
- Source : **CodeCommit**<br>
- Build : **CodeBuild**<br>
- Deploy : **S3**

<br>
정적 컨텐츠의 경우 빌드 과정이 필수가 되지는 않습니다.<br>
npm이나 별도 패키징 및 플러그인을 사용하지 않아 빌드 과정을 생략하는 경우도 있으며, 빌드를 동작시키지 않아도 배포가 가능한 경우도 있습니다.

다만, 대부분의 소스코드들이 소스코드 그 자체만으로 운영서버에서 사용하지 않기 때문에, 소스코드에 대한 패키징 작업이나 후처리 작업을 빌드과정에서 수행해줄 수 있습니다.


<br>
## 실습
---
test

<br>
## 끝으로
---
test


[아키텍쳐 구성도]: http://localhost:4000/assets/img/posts/2021-08-17-codepipeline-frontend-source-cicd_1.png
