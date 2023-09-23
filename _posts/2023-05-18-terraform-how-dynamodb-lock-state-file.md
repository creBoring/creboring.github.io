---
layout: post
title: "[Terraform] DynamoDB의 backend state lock 원리"
summary: "DynamoDB가 S3 에 저장된 tfstate파일을 안전하게 lock 할 수 있는 이유"
author: creboring
date: '2023-05-18 00:20:41 +0530'
category: [테라폼, AWS]
thumbnail: /assets/img/posts/category/terraform_thumbnail.png
title-img: /assets/img/posts/category/blacksmith.jpg
keywords: Terraform, 테라폼, DynamoDB, lock, backend
permalink: /blog/terraform-how-dynamodb-lock-state-file/
usemathjax: true
---

<br>
<!-- excerpt-start -->
여러 작업 환경에서 Terraform 코드를 작성하다보니 tfstate 파일이 잘 관리되지 않아 파일을 원격지에서 관리하게 되었습니다. AWS 인프라를 Terraform으로 작성하던지라 S3와 DynamoDB를 이용해 backend와 lock을 구현하게 되었는데, s3는 이해가 되었지만 dynamodb로 어떻게 lock을 구현한다는건지 의문이 들어 동작 원리를 분석하게 되었습니다. 

## 결론
Terraform은 state 파일에 접근하기 전 DynamoDB에 lock 데이터를 입력하는데, 이 때 **이미 키가 존재할 경우 에러를 반환하도록 조건부 요청으로 Put**을 합니다. 
그러면, 이미 누군가에 의해 DynamoDB에 lock 데이터가 입력된 상태에서, 다른 누군가가 state 파일에 lock을 시도하려고 DynamoDB에 Put을 요청하면 에러가 반환됩니다. (lock 데이터는 모든 작업이 완료된 이후에 삭제합니다.)
> 에러: (ConditionalCheckFailedException: The conditional request failed Lock Info ID: 00000000-0000-0000-0000-000000000000)

이로 하여금 작업 도중 다른 사용자가 lock을 잡을 수 없게 만들어 기존 작업의 lock을 유지합니다.



## 원리
DynamoDB에서 `PutItem`의 기본 동작은 **'덮어쓰기'** 입니다. 동일 키의 항목이 테이블이 이미 존재하는 경우 DynamoDB는 새 데이터로 덮어쓰기를 하며 에러를 반환하지 않습니다. 이를 위해 Terraform에서는 DynamoDB에 lock 데이터를 put 할 때 단순히 PutItem을 하는 것이 아닌 **'조건부 요청'**으로 PutItem 을 수행해 기존 데이터가 있는 경우 **강제로 에러를 반환하도록 합니다**.

### 조건부 요청이란?
AWS는 DynamoDB에서의 CUD(Put, Update, Delete) 는 unconditional, 무조건적이라고 합니다. 해당 요청은 반드시 수행되며, 기존에 데이터가 있는 경우 덮어씁니다.

이를 무조건 성공하게 하지 않고, 특정 경우에만 성공하도록 조건을 걸게하는 옵션이 '조건부 요청' 이라는 옵션입니다. 사용방법은 아래와 같이 CUD 요청에 옵션으로 `condition-expression` 을 주면 됩니다.

```
aws dynamodb put-item 
    --table-name ProductCatalog 
    --item file://item.json 
    --condition-expression "attribute_not_exists(Id)"
```

### 실제로 Terraform에서 수행하는 PutItem 요청
TF_LOG 수준을 DEBUG로 변경하고 terraform에서 `plan`을 수행하면 아래와 같은 로그를 찾아볼 수 있습니다.
```
[DEBUG] [aws-sdk-go] DEBUG: Request dynamodb/PutItem Details:
---[ REQUEST POST-SIGN ]-----------------------------
POST / HTTP/1.1
Host: dynamodb.ap-northeast-2.amazonaws.com
User-Agent: APN/1.0 HashiCorp/1.0 Terraform/1.4.5 aws-sdk-go/1.44.122 (go1.19.6; windows; 386)
Content-Length: 427
Accept-Encoding: identity
Authorization: AWS4-HMAC-SHA256 Credential=00000000000000000000000/20230101/ap-northeast-2/dynamodb/aws4_request, SignedHeaders=accept-encoding;content-length;content-type;host;x-amz-date;x-amz-target, Signature=00000000000000000000000000000000000000000000000000000000000000
Content-Type: application/x-amz-json-1.0
X-Amz-Date: 20230101T000000Z
X-Amz-Target: DynamoDB_20120810.PutItem

{"ConditionExpression":"attribute_not_exists(LockID)","Item":{"Info":{"S":"{\"ID\":\"00000000-0000-0000-0000-000000000000\",\"Operation\":\"OperationTypePlan\",\"Info\":\"\",\"Who\":\"DESKTOP-0000000\\\\crebo@DESKTOP-0000000\",\"Version\":\"1.4.5\",\"Created\":\"2023-01-01T00:00:00.0000000Z\",\"Path\":\"test-tfstate/terraform.tfstate\"}"},"LockID":{"S":"test-tfstate/terraform.tfstate"}},"TableName":"terraform-lock"}
-----------------------------------------------------
```

위에서 주목해야할 옵션은 아래 라인으로
```
"ConditionExpression":"attribute_not_exists(LockID)"
```

조건에 **"attribute가 존재하지 않을 때"** 라는 조건을 주었기 때문에, 누군가 작업하고 있을 경우 다른 사람이 lock을 시도하면 에러가 반환됩니다.


## 참고 자료
- <small>내용 참고 / <a href="https://docs.aws.amazon.com/ko_kr/amazondynamodb/latest/developerguide/WorkingWithItems.html#WorkingWithItems.ConditionalUpdate" target="_blank">AWS Doc - [DynamoDB] Working with items and attributes</a></small>
- <small>사진 출처(Freepik) / <a href="https://www.freepik.com/free-vector/blacksmith-with-hard-work-strength-symbols-flat-illustration_15329625.htm#query=blacksmith&from_query=smithy&position=1&from_view=search&track=sph" target="_blank">blacksmith with hard work - macrovector</a></small>