---
layout: post
title: "[Terraform] 테라폼 클라우드 도입기"
summary: "Terraform Cloud를 활용하여 리소스 편하게 관리하기"
author: creboring
date: '2023-09-23 17:00:34 +0900'
category: [테라폼, 테라폼 클라우드]
thumbnail: https://media.giphy.com/media/woD3lyw4gsMBJgHlpN/giphy.gif
title-img: /assets/img/posts/category/blacksmith.jpg
keywords: Terraform, 테라폼, terraform cloud, cloud, aws, 테라폼 클라우드, 클라우드
permalink: /blog/terraform-cloud-intro/
usemathjax: true
---

<figure>
    <img src="https://media.giphy.com/media/xT9IgG50Fb7Mi0prBC/giphy.gif" class="img-fluid">
    <figcaption><small>안녕하세요</small></figcaption>
</figure>


최근 팀 인프라 리소스를 정리하는 과정에서 Terraform과 Terraform Cloud를 도입하게 되었습니다. 그 중에서도 이번 글에서는 **Terraoform Cloud**를 도입하며 경험했던 다양한 기능들과 문제 해결 방안들에 대해 중점적으로 공유하고자 합니다.

## 도입 배경...
<!-- excerpt-start -->
현재 저희 팀은 다른 팀들과 한 AWS 계정을 공유하며, 필요한 리소스를 각각 생성해 유지 관리하고 있습니다. 그러다보니 각 팀마다 사용중인 리소스의 비용 추적이 어렵고, 리소스가 늘어날수록 네트워크, 스토리지와 같은 자원 관리에 어려움이 생기기 시작했습니다.

결국 팀에서 쓰던 AWS 리소스들을 별도의 새로운 AWS 계정으로 분리하게 되었는데, 기존 리소스들의 형상 관리가 잘되고 있지 않아 옮겨야할 리소스 현황 파악이 상당히 어려웠습니다. (여러 팀들 돌아다니며 리소스 주인 찾기...)

<figure>
    <img src="/assets/img/posts/2023-09-23/aws_resource_capture.png" class="img-fluid">
    <figcaption><small>어느 팀에서 사용중인지 모를 temp 리소스들..</small></figcaption>
</figure>

게다가 환경별, 도메인별 중복되는 리소스들은 python으로 구현해 일괄 생성되도록 구성해두긴 했으나, 실행된 리소스에 대한 상태 체크가 어렵고, 실수로 잘못 배포한 경우엔 롤백이 어려워 인프라 관리 개선을 위한 방안이 필요했습니다.

## 그렇게 시작하게 된 IaaC

<figure>
    <img src="https://media.giphy.com/media/woD3lyw4gsMBJgHlpN/giphy.gif" class="img-fluid">
</figure>

이를 개선하기 위해 IaaC를 도입하는 것은 선택이 아닌 필수였습니다. IaaC를 도입함에 있어 여러 툴들이 옵션으로 있었지만 (Pulumi, CloudFormation, ...), 러닝커브가 적은 선언형의 언어를 사용한다는 점, 상세한 Docs와 오픈소스 기반 다양한 커뮤니티가 잘 활성화 되어있는 점을 고려하여 **Terraform**을 선택하게 되었습니다. 

비슷한 선택지로 **AWS CloudFormation**이 유력한 후보에 있었으나, 상용 서비스인만큼 다른 오픈소스 라이브러리들과의 호환성이나 자유도가 부족하다 느꼈고, 제공하고 있던 인터페이스도 테라폼을 제치고 선택할만큼 만족스럽진 못하다 판단했습니다.

결국 선택의 승자가 된 테라폼을 이용해 열심히 코드를 작성하던 도중 또 다른 문제에 도달하게 되는데... (~~이로 인해 작성중인 모듈 구조를 싹 바꿔버리게 되었다는 건 안비밀~~)
<!-- 이 선택에 따라 팀 내 개발자들은 앞으로 Terraform 코드를 작성해 리소스를 배포하고 관리하게 되었는데, Terraform의 경우 각 리소스에 대한 상태 정보는 한 곳에서 관리되어야하기 때문에, 여러 환경에서 제한 없이 코드가 실행되어서는 안됩니다. -->

## 모듈의 버저닝과 협업 문제
테라폼 코드를 처음 작성하면서 가장 어려웠던 점은 모듈 구조 구성이였습니다. 여러 Best Practice와 블로그 글들을 참고하며 모듈 구조를 짜다보니 **모듈의 버저닝**이 가능하다는 것을 알게되었고, 각 환경마다 다른 버전을 사용하게 하여, 운영 소스에 영향 없이 개발계에서 원활히 테스트를 수행하고자 버저닝을 도입하기로 했습니다.

그런데 이 버저닝을 사용하기 위해선, 해당 모듈을 local이 아닌 모듈 레지스트리에서 가져와야만 했습니다. 저희는 `workspace` → `stack` → `module` 구조로 모듈 구성을 가져갔기 때문에, stack과 module을 모두 레지스트리에 저장해야했습니다.

<figure>
    <img src="/assets/img/posts/2023-09-23/terraform_cloud_module_architect.png" class="img-fluid">
    <figcaption><small>대략적으로 구상한 모듈 구조의 샘플...</small></figcaption>
</figure>

이 때 처음 **Terraform Cloud**를 접하게 되었는데, 모듈 레지스트리를 구성해 버저닝하기 위해선 방금 언급한 Terraform Cloud에 private 레지스트리를 구성하거나, 별도의 source 레지스트리에서 지원하는 방식으로 버전을 명시하는  source url을 사용하는 방법이 있었습니다.

그리고 또 다른 문제점은, 리소스를 선언해놓은 테라폼 코드가 개발자의 각 PC에서 실행되기 때문에, `개발자 A`가 git에 push한 테라폼 소스코드를, `개발자 B`가 pull 하지 않은 상태로 자신의 소스코드를 실행할 수 있어, 자칫하면 리소스가 롤백될 수 있는 치명적인 문제가 존재했습니다.

현재 생성된 리소스의 상태 파일은 **S3**에 저장하고 **dynamodb**로 locking 되어 있어 안전하긴 하지만, 이는 동시 실행만 방지할 뿐 소스코드의 충돌이나 최신화를 강제할 수는 없습니다.

## 그렇게 도입된 Terraform Cloud
위 두 가지 문제점을 해결하기 위한 방안으로 **Terraform Cloud**와 **Atlantis** 두 가지 툴이 고려되었습니다. 두 도구 모두 위에서 언급한 협업 문제를 해결할 수 있지만, Atlantis의 경우 Git을 사용한 협업 워크플로우에 초점이 맞춰진 반면, Terraform Cloud의 경우 테라폼 사용 경험 전반에 걸쳐 다양한 기능들을 제공하기 때문에, 여러 기능들을 테스트해본 결과 테라폼 클라우드를 선택하게 되었습니다.

> 그리고 Atlantis를 사용할 경우 별도 실행 서버를 구동해줘야하며, 위 언급된 모듈의 버저닝 문제는 Git 자체 기능을 통해 해결해주어야 합니다. 인원이 적은 저희 팀 구조에는 Terraform Cloud가 더 적합하다 판단했습니다.

**Terraform Cloud**를 선택하게 만든 Terraform Cloud의 powerful한 기능들은 아래와 같습니다.
- 테라폼 모듈 버저닝 지원
- Terraform Cloud의 별도 backend state 저장소 제공
- 웹 콘솔로 즉시 이용 가능, 구축이나 설정이 필요없음. 거기다 무료 (물론 유료 플랜도 있음)
- 여러 VCS와 연동하여 자동 플래닝 및 플랜의 상태 공유 가능
- 조직을 만들어 팀원들과 함께 접근 가능 및 각 팀원별 권한 설정 가능
- SSO IDP 지원으로 간편히 AWS 자격증명 관리

이 뿐만 아니라 Global Variable을 설정하여 공통으로 적용될 variable을 지정하거나, tfe output을 지정하여 특정 workspace의 output을 다른 workspace의 input으로 사용 가능한 점 등, 인프라 구성에 있어 유용한 기능들이 매우 많았습니다.

이 중에서도 특히 유용했던 몇몇 기능들의 경험에 대해서 자세히 공유드리겠습니다.

### 자동 플래닝과 상태 확인

<figure>
    <img src="/assets/img/posts/2023-09-23/terraform_cloud_run_result.png" class="img-fluid">
</figure>

Terraform Cloud를 사용하면 VCS와 연계하여, 변경사항에 대해 자동으로 `Plan` → `Apply` 까지 수행할 수 있습니다. 저희 팀에서는 물론 PR 요청을 검토하고 있지만, Plan에서 Apply 까지 고속도로가 뚫리는 것은 위험하다 판단하여, 우선 Plan만 수행되고 해당 결과에 따라 코멘트를 남기거나 Apply를 수행해주고 있습니다.

<figure>
    <img src="/assets/img/posts/2023-09-23/terraform_cloud_plan_result.png" class="img-fluid">
</figure>

위와 같이 플랜이 성공하고, 목적에 맞는 리소스들만 변경되는지 확인되었으면, Web UI상 버튼을 통해 Apply를 실행시켜줍니다.

### Private Registry

<figure>
    <img src="/assets/img/posts/2023-09-23/terraform_cloud_registry.png" class="img-fluid">
</figure>

위에서도 언급했듯이, 모듈의 버저닝을 위해서는 이를 레지스트리에 저장하고 소스로 지정하여 사용해야합니다. Terraform Cloud에서는 프로젝트에서 사용할 모듈을 import 또는 선택할 수 있는데, 이를 이용해 VCS에 저장된 사설 레포지토리의 모듈도 쉽게 import하여 사용할 수 있습니다.

<figure>
    <img src="/assets/img/posts/2023-09-23/terraform_cloud_registry_add.png" class="img-fluid">
</figure>

모듈의 레포지토리명은 `terraform-<provider>-<name>` 형식을 갖춰야하며, 위 사진처럼 VCS와 레포지토리를 선택해 사설 모듈로 등록할 수 있습니다.
또한, 필요한 모듈이 있을 경우 public registry에 등록된 모듈들도 찾아서 추가할 수 있어 편리하게 사용가능합니다.

<figure>
    <img src="/assets/img/posts/2023-09-23/terraform_cloud_registry_info.png" class="img-fluid">
</figure>

저의 경우 필요 리소스 구성이 있을 때, 위 public registry에서 먼저 공식 모듈들을 찾아보고, 만약 없다면 사설 VCS에 위와 같이 private registry를 추가하여 사용했습니다. git의 경우 release number를 통해 버저닝이 지원되기 때문에 간편하게 버저닝을 이용할 수 있었습니다. (릴리즈 넘버링 규칙은 `x.y.z` 이여야 합니다.)

<figure>
    <img src="/assets/img/posts/2023-09-23/terraform_module_usage_example.png" class="img-fluid" width="400px">
    <figcaption><small>소스 위치 정보는 Terraform Cloud에 레지스트리 추가 시 생성됩니다</small></figcaption>
</figure>

모듈을 사용할 땐 위와 같이 소스 정보와 버전을 입력하면 됩니다.

### SSO를 통한 AWS 자격증명 관리

Terraform 코드를 실행할 때 provider의 credential을 소스코드 안에 주입시킬 수도 있지만, 이는 보안적으로 위험한 방법입니다. 보통은 Terraform 코드를 실행하는 서버 자체에 credential을 세팅하는 경우가 많은데, Terraform Cloud의 경우 **OIDC 방식의 SSO IDP**를 제공하기 때문에, 이를 provider와 연결하여 안전하게 자격증명을 관리할 수 있습니다.

제가 사용한 AWS provider의 경우 권한을 부여할 IAM Role을 생성하고, IAM Identity providers에서 IDP 공급자로 `app.terraform.io` 를 지정했습니다.

<figure>
    <img src="/assets/img/posts/2023-09-23/aws_iam_identity_providers.png" class="img-fluid">
</figure>

이후 아래와 같이 workspace의 env로 해당 role arn 과 flag를 추가해주면, Terraform Cloud가 코드를 실행시킬 때 전달받은 role을 위임받아 리소스를 구성합니다.

<figure>
    <img src="/assets/img/posts/2023-09-23/terraform_cloud_sso.png" class="img-fluid">
</figure>

자세한 구성 방법은 아래 공식문서를 참고하시면 방법이 상세히 나와있습니다.
> https://developer.hashicorp.com/terraform/cloud-docs/users-teams-organizations/single-sign-on

### Variable Sets
Terraform으로 리소스들을 구성할 때, 서로간의 간섭을 최소화하기 위해 workspace와 상태 파일을 여러개 구성하는 경우가 있습니다. 이 때, 모든 workspace에서 공통적으로 사용되는 variable이나 env가 있다면 이를 각각 지정해주어야 하는데, Terraform Cloud의 **variable sets**를 이용하면 이 값들을 하나의 set로 관리할 수 있습니다.

<figure>
    <img src="/assets/img/posts/2023-09-23/terraform_cloud_variable_set.png" class="img-fluid">
    <figcaption><small>전체 workspace에서 공통적으로 사용되는 variable들</small></figcaption>
</figure>

또한 Terraform Cloud의 경우 **Project** 라는 **논리적인 workspace 그룹**을 제공하고 있습니다. 각 project별로 다른 variable set를 설정할 수도 있기 때문에, 운영계와 개발계같은 환경 단위로 같은 key의 variable에 다른 값을 대입하도록 구성할 수 있어, 보다 손쉽게 여러 환경을 구성할 수 있습니다.

<figure>
    <img src="/assets/img/posts/2023-09-23/terraform_cloud_projects.png" class="img-fluid">
    <figcaption><small>project라는 논리 단위를 구성하고 하위에 여러 workspace를 구성할 수 있습니다</small></figcaption>
</figure>

저는 위 보시는 것과 같이 `PRD ENV`, `DEV ENV`, `GLOBAL`, `MGMT` 와 같은 논리 그룹을 만들고, 하위에 각 그룹에 필요한 workspace들을 분리하여 배치했습니다. ex) network, data-storage, app, ...

그리고 테라폼으로 구성하는 리소스의 prefix에 환경명을 입력해주었는데, 이 또한 variable set로 구성하여 운영 variable set는 prd project에 부여하고, 개발 variable set는 dev project에 부여하는 방식으로, 소스코드는 동일한 variable key를 사용하지만 입력된 값은 환경별로 달라지게 구성했습니다.

> Projects 개념은 비교적 최근에 발표된 개념으로 위와 같은 variable뿐만 아니라, Team 협업으로써 각 유저의 권한 부여 단위로도 사용가능한 점 등, 다양한 옵션들이 생겨났습니다.<br>
https://www.hashicorp.com/blog/terraform-cloud-adds-projects-to-organize-workspaces-at-scale

## 도입 결과
Terraform을 이용해 기존 리소스를 새로운 계정에 구성하는 것이 모두 완료되었습니다. Terraform Cloud에 구성해둔 리소스들은 팀원분들을 초대해 내역을 함께 공유했으며, `workspace` → `stack` → `module` → `resource` 로 구성된 단계적 모듈화는 버저닝을 통해 기존 운영중인 리소스에 영향 없이, 신규 옵션을 추가하거나 새로운 리소스를 생성할 수 있게 되었습니다. `+1`

또한, 도메인 특성에 의해 빈번히 수동으로 만들어주던 Athena Table들도 Terraform으로 모듈화하여 한번에 구성 및 관리할 수 있게 되었으며, 각 개발자마다 리소스 Stack을 이용하여 자신만의 테스트 환경을 손쉽게 만들 수 있게 되었습니다. `+2`

그리고 이 이력들은 모두 Git에 의해 관리되며, PR을 통해 내부 개발자들간에 리뷰를 거쳐 구성되기 때문에, 더이상 리소스에 대한 변경을 추적을 위해 CloudTrail을 분석하지 않아도 되게 되었습니다. `+3`

마지막으로 Terraform을 통해 리소스에 태깅 또한 가능했기에, 태그 기반의 비용 할당 태그를 통한 AWS 비용 추적도 용이해졌습니다. `+4`

아무래도 Terraform이라는 선언형 언어의 특성상 단순하고 이해하기는 쉽지만 구조적인 코드를 작성하기 어려웠는데, Terraform Cloud의 여러 기능을 활용하니 도메인 정보나 서비스 로직 외적인 부분들을 코드로부터 분리할 수 있었습니다.

그렇게 구성된 리소스는 곧 있을 트래픽 마이그레이션 작업만을 앞두고 있습니다 😎

## 후기

<figure>
  <img src="https://media.giphy.com/media/xUPGcg1IJEKGCI6r5e/giphy.gif" class="img-fluid">
  <figcaption><small>무한정 감사</small></figcaption>
</figure>

팀 내에 새로운 기술을 도입함에 있어 인지 부하가 높아질 수 있는 점을 고려하지 않을 수 없었는데, 다행히도 Terraform을 도입함에 있어 긍정적인 반응을 내준 팀원들께도 감사를 표합니다.

앞으로 테라폼으로 구성된 인프라를 운영하며, 여러 새로운 기능을 알게되거나 개선해나갈 부분들이 무궁무진하게 있을 것으로 예상되고, 또 이런 점들이 신나기도 합니다. 그런데 벌써, 테라폼의 라이센스 정책이 바뀌는가 하면, OpenTofu 라는 오픈소스가 fork 되는 둥 엄청 많은 변화들을 직격으로 맞고 있습니다... (~~이 정도로 변하는 건 원치 않았는데...~~)

모두들 테라폼으로 인프라 구성을 코드화하여 운영에 평화가 찾아오기를 기원합니다 🙇🏻‍♂️

## 도움이 되었던 자료들
아래 자료들은 Terraform과 Terraform Cloud를 도입함에 있어 정말 크게 도움이되었던 자료들입니다. 자료 제공자분들께 사랑과 감사를 표합니다. (기억력이 좋지 않아, 기억에 남는 자료들만 공유합니다...)
- <small><a href="https://youtu.be/m9HeYtzeiLI" target="_blank">당근마켓 박병진님의 유튜브 발표자료 - 확장 가능한 테라폼 코드 관리</a></small>
- <small><a href="https://youtu.be/3qSpwqckvXQ" target="_blank">AWS Container Hero 송주영님의 유튜브 자료 - ... AWS Hero 가 이야기하는 Terraform 기본설명</a></small>
- <small><a href="https://helloworld.kurly.com/blog/terraform-adventure/" target="_blank">컬리 기술블로그 - DevOps팀의 Terraform 모험</a></small>


