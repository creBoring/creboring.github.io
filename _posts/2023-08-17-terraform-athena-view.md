---
layout: post
title: "[Terraform] 테라폼으로 Athena View 만드는 법"
summary: "Terraform Resource에 Glue Catalog Table만 있고 View는 없는 이유"
author: creboring
date: '2023-08-17 02:30:22 +0530'
category: [테라폼, AWS]
thumbnail: /assets/img/posts/category/terraform_x_athena.png
title-img: /assets/img/posts/category/blacksmith.jpg
keywords: Terraform, 테라폼, glue, athena, table, view, aws, presto
permalink: /blog/terraform-athena-view/
usemathjax: true
---

<figure>
    <img src="/assets/img/posts/category/terraform_x_athena.png" class="img-fluid">
</figure>

<!-- excerpt-start -->
테라폼으로 Glue Catalog 테이블 및 View를 만드려고 공식 문서에 들어가보면, 테이블에 대한 사용법만 있고, View에 대해서는 아무런 사용 방법도 명시되어 있지 않습니다. 테이블과 달리 View의 경우 복잡한 쿼리문이나 Join문이 쿼리에 포함될 수도 있는데, 이를 테라폼으로 구현하는 방법에 대해 정리해보았습니다.

## Terraform 리소스 
Terraform으로 Glue Catalog View 리소스를 생성할 때는 **aws_glue_catalog_table** 리소스를 선택하고 몇 가지 옵션을 제공해야 합니다. aws_glue_catalog_table 리소스를 생성할 때 사용 가능한 옵션들 중 View와 관련된 옵션들은 아래와 같습니다.
- `table_type`: Table or View 선택
- `Parameters`: Glue Table의 파라미터로, View의 경우 `presto_view` 라는 값을 true 로 넣어주어야 합니다.
- `view_expanded_text`: `"/* Presto View */"` 라는 고정된 값을 가집니다.
- `view_original_text`: View 생성에 필요한 여러 정보를 text로 담고 있습니다.

table_type은 어떤 옵션인지 바로 유추 가능하지만, 아래 두 옵션들은 직관적이지 않습니다. 이는 **Presto** 엔진의 View에서 사용하는 옵션들로, 주석 형태의 String을 넘겨줌으로써 View를 만들 때 필요한 정보를 전달받습니다. 주석 안에는 base64로 인코딩된 Object 정보를 넘겨줘야 합니다. (Athena는 Trino 및 Presto 엔진 기반으로 제공되는 관리형 서비스입니다.)

추가로 View 리소스를 만들 때는 `SerdeInfo` 항목을 만들지 않거나 빈 Map을 제공해야 합니다.

### Example) Terraform 코드의 일부분..
```hcl
resource "aws_glue_catalog_table" "glue_datacatalog_tables" {
  name          = "test_table"
  database_name = "test_db"

  table_type         = "VIRTUAL_VIEW" 
  view_expanded_text = "/* Presto View */"
  view_original_text = ("/* Presto View: ${base64encode(jsonencode({
    "catalog" : "test_catalog",
    "schema" : "test_db",
    "columns" : var.columns,
    "originalSql" : var.originalSql
  }))} */")
}

```

## Presto View 옵션
Athena로 데이터를 쿼리하게 되면 메타데이터 정보를 Glue를 통해 가져오게 되는데, 만약 쿼리하려는 테이블이 View 라면 Glue Table에 저장된 view_original_text 값에서 필요한 정보들을 추출해 데이터를 쿼리하게 됩니다. 

view_original_text에는 어떠한 정보를 넘겨주어야되는지 Athena 서비스의 근간이 되는 Presto 오픈소스 코드를 분석해보았습니다.

### view_original_text 포맷
view_original_text은 정해진 형식의 주석을 prefix, suffix로 지정해주어야 합니다.
- prefix: 테이블이 View인지 판별할 때 `/* Presto View:` 라는 prefix가 사용됩니다.
- suffix: 주석을 닫아주기 위해 `*/` 가 사용됩니다.

다 작성이 되고 나면 다음과 같은 포맷의 텍스트가 완성되어야 합니다. 
```
/* Presto View: (base64 인코딩된 ConnectorViewDefinition 정보) */
```

(ConnectorViewDefinition 정보 앞뒤로 공백을 하나씩 두는게 좋습니다)

> presto 엔진의 encodeViewData 메소드를 참고하세요.<br>
<a href="https://github.com/prestodb/presto/blob/fea80c96ddfe4dc42f79c3cff9294b88595275ce/presto-hive/src/main/java/com/facebook/presto/hive/HiveUtil.java#L741">https://github.com/prestodb/presto/blob/fea80c96ddfe4dc42f79c3cff9294b88595275ce/presto-hive/src/main/java/com/facebook/presto/hive/HiveUtil.java#L741</a>

### ConnectorViewDefinition
definition 정보는 아래와 같은 정보를 담고 있어야 합니다. **catalog**, **schema**의 경우 테이블 만들때 명시해둔 값을 똑같이 넣어두면 되며, **originalSql**에는 View를 생성하는 쿼리, **columns**에는 View 테이블에서 사용할 컬럼들의 이름과 타입을 명시하면 됩니다. (View 생성 쿼리에 의해 컬럼의 타입이 변경되었다면 이 또한 맞춰주어야 합니다)
- catalog: 카타로그 이름
- schema: 스키마 이름
- columns: View에서 사용하는 컬럼 리스트
- originalSql: View를 생성하는 Select 쿼리문

> presto 엔진의 ViewDefinition 클래스를 참고하세요.<br>
<a href="https://github.com/prestodb/presto/blob/master/presto-spi/src/main/java/com/facebook/presto/spi/analyzer/ViewDefinition.java#L26">https://github.com/prestodb/presto/blob/master/presto-spi/src/main/java/com/facebook/presto/spi/analyzer/ViewDefinition.java#L26</a>

> 위 presto의 ViewDefinition 클래스를 보면 위 4개 컬럼 외에도 owner라는 컬럼도 필요로 하는 것으로 보이는데, Athena로 View를 만들어 properties를 살펴보면 해당 항목이 존재하지 않습니다... 왜 그런 차이가 있는지에 대해선 조금 더 분석이 필요할 듯 하네요

위와 같은 정보를 json형식대로 만들어보면 아래와 같은 형태가 나옵니다. 이를 view_original_text 포맷에 맞게 넣어주면 됩니다.
```json
{
    "catalog" : "test_catalog",
    "schema" : "test_db",
    "columns" : [  
      {
        "name": "컬럼_1",
        "type": "varchar"
      },
      {
        "name": "컬럼_2",
        "type": "decimal(20,10)"
      }
    ],
    "originalSql" : "SELECT \"column_1\" \"컬럼_1\", CAST(\"column_2\" AS decimal(20, 10)) \"컬럼_2\" FROM test_table"
  }
```

> **※ view_original_text의 column에는 string이 허용되지 않습니다. string은 모두 varchar 로 변환하여 입력해주어야 합니다.**

## Example
위 분석한 구조대로 실제 Terraform 코드를 작성해 Table과 View를 만들어보겠습니다.

### Glue Catalog Table
일반적인 테이블은 Terraform 공식문서의 예시만 참고해도 쉽게 만들 수 있습니다. View를 만들기 이전에, 간단한 예제를 통해 테이블을 먼저 만들어보겠습니다.

```hcl
resource "aws_glue_catalog_table" "test_table" {
  name          = "user_table"
  database_name = "db"
  table_type    = "EXTERNAL_TABLE"

  parameters = {
    "EXTERNAL"           = "TRUE"
    "has_encrypted_data" = "false"
    "auto.purge"         = "true"
  }

  storage_descriptor {
    location      = "s3://my-bucket/event-streams/my-stream"
    input_format  = "org.apache.hadoop.mapred.TextInputFormat"
    output_format = "org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat"

    ser_de_info {
      name                  = "stream"
      serialization_library = "org.apache.hadoop.hive.serde2.OpenCSVSerde"
      parameters            = {
        "serialization.format" = 1
      }
    }

    columns {
      name = "name"
      type = "string"
    }

    columns {
      name = "age"
      type = "string"
    }
  }
}
```

위 리소스를 생성하면, `name`, `age` 두 컬럼을 가진 user 테이블이 만들어집니다. 이제 age를 int로 변환하는 View를 만들어보겠습니다.

### Glue Catalog View
View의 경우 위에서 언급한 view_original_text 값으로 ConnectorViewDefinition에 필요한 정보를 담은 Object를 만든 후 base64로 인코딩하여 prefix, suffix를 추가해주었습니다.

```hcl
resource "aws_glue_catalog_table" "test_view" {
  name          = "user_view"
  database_name = "db"
  table_type    = "VIRTUAL_VIEW"

  view_params = {
    "presto_view" = "true"
    "comment"     = "Presto View"
  }
  view_expanded_text = "/* Presto View */"
  view_original_text = "/* Presto View: ${base64encode(jsonencode({
    "catalog" : "catalog_name",
    "schema" : "db_name",
    "columns" : [  
      {
        "name": "name",
        "type": "varchar"
      },
      {
        "name": "age",
        "type": "tinyint"
      }
    ],
    "originalSql" : "SELECT \"name\", CAST(\"age\" AS tinyint) \"age\" FROM user_table"
  }))} */"

  storage_descriptor {
    input_format  = "org.apache.hadoop.mapred.TextInputFormat"
    output_format = "org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat"

# View의 경우 ser_de_info 정보를 제공하지 않습니다.
#    ser_de_info {}

    columns {
      name = "name"
      type = "string"
    }

    columns {
      name = "age"
      type = "tinyint"
    }
  }
}
```

## 정리 후기
AWS 리소스들을 Terraform으로 마이그레이션하면서, 기존에 있던 Athena 테이블 정보들도 옮기게 되었는데 View를 만드는 방법이 잘 설명되어 있지 않아 좀 헤맸던것 같네요. 

관련해서 각 옵션들이 어떻게 쓰이는지 알아보다보니 Terraform이 아닌 Presto, Trino 엔진에 대해 공부하게 되는... 상당히 복잡하면서도 신기한 경험이였습니다.

## 참고자료
- <small><a href="https://stackoverflow.com/questions/56289272/create-aws-athena-view-programmatically/56347331#56347331" target="_blank">https://stackoverflow.com/questions/56289272/create-aws-athena-view-programmatically/56347331#56347331</a></small>
- <small><a href="https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/glue_catalog_table" target="_blank">https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/glue_catalog_table</a></small>
- <small><a href="https://github.com/prestodb/presto">https://github.com/prestodb/presto</a></small>