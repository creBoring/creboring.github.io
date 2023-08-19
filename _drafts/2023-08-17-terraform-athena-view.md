---
layout: post
title: "[Terraform] 테라폼으로 Athena View 만드는 법"
summary: "Terraform Resource에 Glue Catalog Table만 있고 View는 없는 이유"
author: creboring
date: '2023-08-15 02:30:22 +0530'
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

테라폼으로 Glue Catalog 테이블 및 View를 만드려고 공식 문서에 들어가보면, 테이블에 대한 사용법만 있고, View에 대해서는 아무런 사용 방법도 명시되어 있지 않습니다. 테이블과 달리 View의 경우 복잡한 쿼리문이나 Join문이 쿼리에 포함될 수도 있는데, 이를 테라폼으로 구현하는 방법에 대해 분석해보았습니다.

## Terraform 리소스 
Terraform으로 Glue Catalog View 리소스를 생성할 때는 **aws_glue_catalog_table** 리소스를 선택하고 몇 가지 옵션을 제공해야 합니다. aws_glue_catalog_table 리소스를 생성할 때 사용 가능한 옵션들 중 View와 관련된 옵션들은 아래와 같습니다.
- `table_type`: Table or View 선택
- `view_expanded_text`: `"/* Presto View */"` 라는 고정된 값을 가집니다.
- `view_original_text`: View 생성에 필요한 여러 정보를 text로 담고 있습니다.

table_type은 어떤 옵션인지 바로 유추 가능하지만, 아래 두 옵션들은 직관적이지 않습니다. 이는 **Presto** 엔진의 View에서 사용하는 옵션들로, 주석 형태의 String을 넘겨줌으로써 View를 만들 때 필요한 정보를 전달받습니다. 주석 안에는 base64로 인코딩된 Object 정보를 넘겨줘야 합니다. (Athena는 Presto 엔진 기반으로 제공되는 관리형 서비스입니다.)

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

### view_original_text 작성법
view_original_text의 포맷은 `/* Presto View: ` + (base64 인코딩된 definition 정보) + `*/` 를 입력해주면 됩니다. (def 정보 앞뒤로 공백을 하나씩 두는게 좋습니다)

definition 정보는 아래와 같은 정보를 담고 있어야 합니다. **catalog**, **schema**의 경우 테이블 만들때 명시해둔 값을 똑같이 넣어두면 되며, **originalSql**에는 View를 생성하는 쿼리, **columns**에는 View 테이블에서 사용할 컬럼들의 이름과 타입을 명시하면 됩니다. (View 생성 쿼리에 의해 컬럼의 타입이 변경되었다면 이 또한 맞춰주어야 합니다)
- catalog
- schema
- columns
- originalSql

> presto 엔진의 encodeViewData 메소드를 참고하세요.<br>
https://github.com/trinodb/trino/blob/27a1b0e304be841055b461e2c00490dae4e30a4e/presto-hive/src/main/java/io/prestosql/plugin/hive/HiveUtil.java#L597-L600


## Example
실제로 Terraform으로 Table과 View를 만들어보며 위 논리대로 잘 구성되는지 살펴보겠습니다.

### Glue Catalog Table
일반적인 테이블은 Terraform 공식문서의 예시만 참고해도 쉽게 만들 수 있습니다. View를 만들기 이전에, 간단한 예제를 통해 테이블을 먼저 만들어보겠습니다.

```hcl
resource "aws_glue_catalog_table" "test_table" {
  name          = "user"
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
View의 경우 Terraform에서 aws_glue_catalog_table 를 생성할 때, 몇 가지 옵션을 제공함으로써 해당 테이블이 View 임을 명시할 수 있습니다.


## 참고자료
- <small><a href="https://stackoverflow.com/questions/56289272/create-aws-athena-view-programmatically/56347331#56347331" target="_blank">https://stackoverflow.com/questions/56289272/create-aws-athena-view-programmatically/56347331#56347331</a></small>
- <small><a href="https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/glue_catalog_table" target="_blank">https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/glue_catalog_table</a></small>