---
title: Terraform IaC 도구를 활용한 AWS 웹콘솔 클릭 노가다 해방기
author: 안재현
tags: 
- terraform
- iac
- aws
---

**테라폼이란?**

HashiCorp 에서 오픈소스로 개발중인, **클라우드 및 온프레미스 인프라를 코드로 관리**할 수 있는 도구입니다. 인프라 환경 구성시 웹 콘솔 등이 아닌 Ansible 같은 선언적 코드형식으로 리소스 들을 생성, 수정, 삭제 하여 관리가 가능한 **IaC (Infrastructure as Code) 프로비저닝 도구**입니다.

HCL 이라는 자체 언어를 사용하여 리소스를 선언하고 AWS, GCP, Azure 와 같은 주요 클라우드 서비스 프로바이더를 선택하여 웹 콘솔에서 마우스 클릭으로 할 수 있는 일들을 코드 배포 과정을 통해 관리할 수 있습니다.

**테라폼 도입 배경**

사람인에서는 현재 아이엠그라운드  모의면접 서비스 외 개발자 채용 플랫폼 점핏, 인사담당자 분들을 위한 HR솔루션 플랩, 1:1 프리랜서 매칭 서비스 사람인긱 등에서 적극적으로 AWS 클라우드 환경을 사용하고 있습니다. 서비스 최초 환경구성 부터 초기 인프라 환경 구성시에는 정해진 일정내에 개발자 분들과 함께 빠르게 요구사항에 맞춰 리소스를 조합하여 업무를 진행하였으나 점차 서비스의 규모가 커지고 복잡해지면서 웹 콘솔 작업만으로는 클릭 노가다 지수가 점차 높아지기 시작하며 향후 작업 히스토리 관리 측면으로 개선점이 필요한 상황이었습니다.

2021년 말 팀내 공론화 과정을 거쳐 2022년 초부터 SRE팀 동료인 박용철님과 함께 테라폼 파일럿을 진행하며 점핏과 플랩 대부분의 AWS 인프라 환경을 코드로서 관리하기 시작하였고 현재까지 개발/스테이징 환경과 운영 환경까지도 테라폼을 활용하여 많은 양의 클릭 노가다가 필요한 업무들로 부터 상당 부분 해방되어 기쁜 마음에 아래와 같이 그 여정을 공유 드립니다.

## 목차

- **테라폼 기본 동작 구조**
- **테라폼 사용준비 첫번째 - AWS IAM 설정**
- **테라폼 사용준비 두번째 - Terraform credentials 환경 설정**
- **테라폼 사용준비 세번째 - Terraform 프로젝트 초기화 설정**
- **테라폼 도입시도 첫번째 - AWS 서비스별 통짜 모듈 TF 작성**
- **테라폼 도입시도 두번째 - AWS 서비스별 모듈 분리 TF 작성**
- **테라폼 도입시도 세번째 - 모듈 분리 TF 장단점**
- **테라폼 도입 후기**

**테라폼 기본 동작 구조**

1. **write** : 인프라 환경을 HCL 언어로 리소스 정의
2. **plan** : 정의된 리소스들을 생성, 수정, 삭제하기 위해 실행 계획을 확인
3. **apply** : 확인된 실행 계획을 리소스 종속성을 고려하여 올바른 순서대로 수행

![terraform1](/img/terraform/terraform_1.png)

테라폼에서는 초기 도입부터 도입 후 지속 사용시에도 위 3가지 단계를 무한 반복하게 됩니다…

AWS 내에서 필요한 서비스들과 하위 리소스 설정들을 모듈화 하여 설정들을 정의하고 (write), 리소스를 추가 생성, 수정, 삭제하기 위한 실행 계획을 확인하고 (plan), 리소스간 종속성을 고려하여 테라폼이 자동으로 실행 순서를 정해서 리소스들을 배포 (apply) 하게 됩니다.

**plan 을 진행하면서 작성된 리소스의 변동사항이나 오류 내용을 정확하게 인지할 수 있고 plan 에 나온 그대로 apply 를 통해 실제 리소스에 반영이 되므로 plan 의 내용을 꼼꼼하게 확인을 하고 apply 를 진행해야 합니다. (중요)**

**테라폼 사용준비 첫번째 - AWS IAM 설정**

테라폼에서 AWS 서비스 리소스들을 접근할 수 있도록 별도의 AdministratorAccess 권한을 가진 IAM 계정과 그룹을 생성합니다.

![terraform2](/img/terraform/terraform_2.png)

![terraform3](/img/terraform/terraform_3.png)

별도의 계정 생성시에는 웹 콘솔 로그인이 아닌 액세스 키를 통해 접근하므로 액세스 키 자격 증명 유형을 선택하여 액세스 키와 시크릿이 생성되도록 합니다.

![terraform4](/img/terraform/terraform_4.png)

**테라폼 사용준비 두번째 - Terraform credentials 환경 설정**

테라폼을 설치 ([https://www.terraform.io/downloads](https://www.terraform.io/downloads)) 하고 cli 를 사용하기 위해 서버 사용자 계정내 아래와 같이 환경 설정을 합니다.

```bash
root # cat /home/ahnjaehyun/.aws/config
[tfadmin_hrlab]
region=us-east-1
output=text

root # cat /home/ahnjaehyun/.aws/credentials 
# tfadmin 의 accesskey 와 secretkey
[tfadmin_hrlab]
aws_access_key_id=IAM tfadmin 계정 보안 자격 증명 참조
aws_secret_access_key=IAM tfadmin 계정 보안 자격 증명 참조
```

위 credentials 설정을 해두고 실제 테라폼 작성을 진행할 디렉토리에서 provider 설정 tf 파일을 작성합니다.

```bash
root # cat 00-required_providers.tf 
terraform {
    required_providers {
        aws = {
            source  = "hashicorp/aws"
            version = "~> 4.8.0"
        }
    }
    required_version = ">= 1.1.6"
}

root # cat 01-provider.tf 
# ~/.aws/config
# ~/.aws/credentials

// default provider
provider "aws" {
    profile = "tfadmin_hrlab"
    region  = "us-east-1"
}

provider "aws" {
    profile = "tfadmin_hrlab"
    region  = "ap-northeast-2"
	alias   = "kr"
}

provider "aws" {
    profile = "tfadmin_hrlab"
    region  = "us-east-1"
    alias   = "us"
}
```

**테라폼 사용준비 세번째 - Terraform 프로젝트 초기화 설정**

먼저 AWS 프로바이더를 선언하였고 여기에 앞서 설정한 credentials 관련 파일들로 접근 가능한 region 을 설정하였습니다. 위 설정을 통해 AWS 에서 설정한 지역을 alias 값을 통해 접근하여 해당 지역에서 접근 가능한 서비스 리소스들을 작성하고 배포할 수 있습니다. 이제 해당 프로젝트 디렉토리에서 초기화 과정을 거칩니다.

```bash
ahnjaehyun $ terraform init
Initializing modules...

Initializing the backend...

Initializing provider plugins...
- Reusing previous version of hashicorp/aws from the dependency lock file
- Reusing previous version of ekristen/pgp from the dependency lock file
- Using previously-installed hashicorp/aws v4.8.0
- Using previously-installed ekristen/pgp v0.2.4

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

terraform init 명령어를 통해 위에서 선언된 AWS 프로바이더 버전으로 플러그인을 설치하여 사용준비 과정을 완료하게 됩니다.

**테라폼 도입시도 첫번째 - AWS 서비스별 통짜 모듈 TF 작성**

테라폼 사용을 위한 준비를 마치고 나면 이곳에서 살게 됩니다..

[https://registry.terraform.io/providers/hashicorp/aws/latest/docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)

네 매뉴얼입니다. 여기에 있는 예제와 표준 양식을 참조하여 나만의 tf 모듈 파일들을 이쁘게 작성해서 잘 쓰면 되는데요.. 이게 참 헬 입니다. 

테라폼 도입 초기에는 이미 웹콘솔을 통해 환경 구성이 되어있는 각 AWS 서비스들을 테라폼 모듈을 작성하여 표준화 관리를 해야하는데 이미 생성된 리소스들을 어떻게 선언해서 코드를 작성할지 막막했습니다. 너무 방대하니까요.

처음에는 IAM, Route53 등 AWS 의 기본 서비스부터 말 그대로 쌩 코딩부터 했습니다. 하기는 하는데 이미 존재하는 리소스들을 선언하여 정의하면 테라폼이 그에 맞는 tfstate 파일을 만들어 주긴 하지만 상세한 옵션 구성까지 완벽하게 맞추기에는 하드코딩이 너무 심할 것 같아 우선 통짜 모듈로 하되, 이미 생성되어 있는 리소스들을 terrform import 기능을 이용하여 가져오며 통짜 모듈 tf 파일을 그대로 하드코딩을 해봤는데 이대로는 아닌 것 같았습니다. 

![terraform5](/img/terraform/terraform_5.png)

(이런식으로 수천개를 언제 다 해…)

**테라폼 도입시도 두번째 - AWS 서비스별 모듈 분리 TF 작성**

**이미 생성된 리소스들을 import 해서 tfstate 파일에 추가를 하되, 서비스별 모듈 TF 를 작성하여 yml 파일 형식으로 설정값을 관리해서 import 를 하고 신규 리소스를 생성하고 apply 시키자!** 

고민 끝에 좀 더 효율적인 방식을 생각해냈지만 서비스별로 표준에 맞게 TF 모듈 작성하는 것도 생각보다 작업양이 만만치 않았습니다. 대부분의 AWS 서비스들은 테라폼으로 관리가 필요한데 이미 생성하여 운영중인 리소스들은 플랩만 해도 천개는 되는데 모듈화 작업 및 설정파일 yml 파일 구성도 고민하면서 같이 하나씩 작성해 나가는 머나먼 여정을 떠나게 되었습니다. 

같은팀 동료인 용철님과 함께 고민하며 모듈의 틀을 잡고 점핏과 플랩 서비스 설정 특성에 맞게 조금씩 옵션을 수정하며 AWS 서비스별 하나씩 하나씩 도장깨기 여정이 시작되었고 아래와 같은 소스 트리 구조가 완성되었습니다.

![terraform6](/img/terraform/terraform_6.png)

맨 왼쪽부터 루트 모듈 TF, 세부 모듈 TF, 모듈별 세팅값 YML, 람다 등에서 사용된 py, html 소스파일.

CloudFront 모듈을 예로들어 루트 모듈은 아래와 같습니다.

```bash
locals {

  cloudfront_vars = {
    for v in ["us-east-1"]:
      v => {
        hrlab = {
          for _v2 in ["distributions", "cachepolicies", "orp"]:
            _v2 => {
              for f in fileset(path.root, format("setting.d/cloudfront/%s/%s/*.yml", v, _v2)):
                trimsuffix(basename(f), ".yml") => try(yamldecode(file(f)), {})
            }
        }
    }
  }

}

#
# CloudFront
#
module "cloudfront" {
  source    = "./modules.d/cloudfront"
  providers = {
    aws     = aws.us
  }
  for_each  = local.cloudfront_vars.us-east-1

  mod_nick  = each.key
  mod_vars  = each.value
  oai_vars  = aws_cloudfront_origin_access_identity.this

  depends_on = [
    aws_cloudfront_origin_access_identity.this,
  ]
}
```

로컬변수로 us-east-1 region 에 있는 yml 설정값에 정의된 패턴에 맞게 for 문을 돌며 값을 가져오게 하고, cloudfront 루트 모듈에서 key, value 에 맞춰서 해당 값들을 세부 모듈 TF 파일에 맞춰서 정의하게 합니다.

```bash
resource "aws_cloudfront_distribution" "this" {
  for_each = local.distributions

  depends_on = [
    aws_cloudfront_cache_policy.this,
  ]
  enabled             = try(each.value.enabled,             true)
  http_version        = try(each.value.http_version,        "http2")
  web_acl_id          = try(each.value.web_acl_id,          try(data.aws_wafv2_web_acl.this[each.value.web_acl_name].arn, null))
  retain_on_delete    = try(each.value.retain_on_delete,    false)
  is_ipv6_enabled     = try(each.value.is_ipv6_enabled,     true)
  price_class         = try(each.value.price_class,         "PriceClass_All")
  comment             = try(each.value.comment,             null)
  tags                = try(each.value.tags,                {})
  default_root_object = try(each.value.default_root_object, null)
  aliases             = try(each.value.aliases,             [])
  restrictions {
    geo_restriction {
      restriction_type = try(each.value.restrictions.geo_restriction.restriction_type, "none")
      locations        = try(each.value.restrictions.geo_restriction.locations,        [])
    }
  }
  viewer_certificate {
    acm_certificate_arn            = try(each.value.viewer_certificate.cloudfront_default_certificate, null) == null ? null : data.aws_acm_certificate.this[each.value.viewer_certificate.acm_domain].arn
    cloudfront_default_certificate = try(each.value.viewer_certificate.cloudfront_default_certificate, true)
    minimum_protocol_version       = try(each.value.viewer_certificate.minimum_protocol_version, "TLSv1.2_2019")
    ssl_support_method             = try(each.value.viewer_certificate.ssl_support_method, "sni-only")
  }
  dynamic "origin" {
    for_each = try(each.value.origin, {})
    iterator = o
    content {
      origin_id           = o.value.origin_id
      origin_path         = try(o.value.origin_path, "")
      domain_name         = o.value.domain_name
      connection_attempts = try(o.value.connection_attempts, 3)
      connection_timeout  = try(o.value.connection_timeout,  10)

      dynamic "s3_origin_config" {
        for_each = length(try(o.value.s3_origin_config, {})) == 0 ? [] : [ o.value.s3_origin_config ]
        iterator = s
        content {
          origin_access_identity = try(s.value.origin_access_identity, local.oai_vars[s.value.oai_name].cloudfront_access_identity_path)
        }
      }

      dynamic "custom_header" {
        for_each = length(try(o.value.custom_header, {})) == 0 ? [] : [ o.value.custom_header ]
        iterator = ch
        content {
          name    = try(ch.value.name, null)
          value   = try(ch.value.value, null)
        }
      }

      dynamic "origin_shield" {
        for_each = length(try(o.value.origin_shield, {})) == 0 ? [] : [ o.value.origin_shield ]
        iterator = sh
        content {
          enabled              = try(sh.value.enabled, false)
          origin_shield_region = try(sh.value.origin_shield_region, null)
        }
      }

      dynamic "custom_origin_config" {
        for_each = length(try(o.value.custom_origin_config, {})) == 0 ? [] : [ o.value.custom_origin_config ]
        iterator = c
        content {
          http_port                = try(c.value.http_port, null)
          https_port               = try(c.value.https_port, null)
          origin_keepalive_timeout = try(c.value.origin_keepalive_timeout, null)
          origin_protocol_policy   = try(c.value.origin_protocol_policy, null)
          origin_read_timeout      = try(c.value.origin_read_timeout, null)
          origin_ssl_protocols     = try(c.value.origin_ssl_protocols, ["TLSv1", "TLSv1.1", "TLSv1.2"])

        }
      }
    }
  }

  dynamic "custom_error_response" {
    for_each = try(each.value.custom_error_response, {})
    iterator = c
    content {
      error_code            = c.value.error_code
      error_caching_min_ttl = try(c.value.error_caching_min_ttl, null)
      response_code         = try(c.value.response_code, null)
      response_page_path    = try(c.value.response_page_path, null)
    }
  }
  default_cache_behavior {
    cache_policy_id = try(each.value.default_cache_behavior.cache_policy_id, false) != false ? each.value.default_cache_behavior.cache_policy_id : (
                          try(regex("^Managed", each.value.default_cache_behavior.sri_cache_policy_id), false) != false ? data.aws_cloudfront_cache_policy.this[each.value.default_cache_behavior.sri_cache_policy_id].id : 
                            aws_cloudfront_cache_policy.this[each.value.default_cache_behavior.sri_cache_policy_id].id
                        )
    origin_request_policy_id = try(each.value.default_cache_behavior.origin_request_policy_id, false) != false ? each.value.default_cache_behavior.origin_request_policy_id : (
                                 try(regex("^Managed", each.value.default_cache_behavior.sri_origin_request_policy_id), false) != false ? data.aws_cloudfront_origin_request_policy.this[each.value.default_cache_behavior.sri_origin_request_policy_id].id : 
                                   try(aws_cloudfront_cache_policy.this[each.value.default_cache_behavior.sri_origin_request_policy_id].id, null)
                               )
    allowed_methods        = try(each.value.default_cache_behavior.allowed_methods, ["GET", "HEAD"])
    cached_methods         = try(each.value.default_cache_behavior.cached_methods, ["GET", "HEAD"])
    target_origin_id       = each.value.default_cache_behavior.target_origin_id
    compress               = try(each.value.default_cache_behavior.compress, true)
    viewer_protocol_policy = try(each.value.default_cache_behavior.viewer_protocol_policy, "redirect-to-https")
    smooth_streaming       = try(each.value.default_cache_behavior.smooth_streaming,       false)
    default_ttl            = try(each.value.default_cache_behavior.default_ttl, 0)
    min_ttl                = try(each.value.default_cache_behavior.min_ttl, 0)
    max_ttl                = try(each.value.default_cache_behavior.max_ttl, 0)

    dynamic "forwarded_values" {
      for_each = length(try(each.value.default_cache_behavior.forwarded_values, {})) == 0 ? [] : [ each.value.default_cache_behavior.forwarded_values ]
      iterator = f
      content {
        query_string = try(f.value.query_string, false)

        cookies {
          forward = try(f.value.cookies.forward, "none")
        }
      }
    }

  }
  dynamic "logging_config" {
    for_each = try(each.value.logging_config, null) == null ? [] : [ each.value.logging_config ]
    iterator = l
    content {
      bucket          = l.value.bucket
      include_cookies = try(l.value.include_cookies, false)
      prefix          = try(l.value.prefix, null)
    }
  }

  dynamic "ordered_cache_behavior" {
    for_each = { for _i, _v in try(each.value.ordered_cache_behavior, []): _i => _v }
    iterator = o
    content {
    cache_policy_id = try(o.value.cache_policy_id, false) != false ? o.value.cache_policy_id : (
                          try(regex("^Managed", o.value.sri_cache_policy_id), false) != false ? data.aws_cloudfront_cache_policy.this[o.value.sri_cache_policy_id].id : 
                            aws_cloudfront_cache_policy.this[o.value.sri_cache_policy_id].id
                        )
    origin_request_policy_id = try(o.value.origin_request_policy_id, false) != false ? o.value.origin_request_policy_id : (
                                 try(regex("^Managed", o.value.sri_origin_request_policy_id), false) != false ? data.aws_cloudfront_origin_request_policy.this[o.value.sri_origin_request_policy_id].id : 
                                   try(aws_cloudfront_cache_policy.this[o.value.sri_origin_request_policy_id].id, null)
                               )
      path_pattern           = try(o.value.path_pattern,    null)
      allowed_methods        = try(o.value.allowed_methods, ["GET", "HEAD"])
      cached_methods         = try(o.value.cached_methods,  ["GET", "HEAD"])
      target_origin_id       = o.value.target_origin_id
      default_ttl            = try(o.value.default_ttl, 0)
      min_ttl                = try(o.value.min_ttl, 0)
      max_ttl                = try(o.value.max_ttl, 0)
      compress               = try(o.value.compress, true)
      viewer_protocol_policy = try(o.value.viewer_protocol_policy, "redirect-to-https")
      dynamic "forwarded_values" {
        for_each = length(try(o.value.forwarded_values, {})) == 0 ? [] : [ o.value.forwarded_values ]
        iterator = f
        content {
          query_string         = try(f.value.query_string, false)
          headers              = try(f.value.headers, ["Origin"])
          cookies {
            forward            = try(f.value.cookies.forward, "none")
          }
        }
      }
    }
  }

}
```

세부 CF 모듈에서는 위와 같이 aws_cloudfront_distribution 리소스에 대한 세부 정의를 매뉴얼 ([https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudfront_distribution](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudfront_distribution)) 에 맞게 정의를 하고, 모듈별 세팅 YML 파일에서는 key value 형식으로 세부 설정값을 정의한 뒤 아래처럼 import 를 합니다.

```bash
$ terraform import aws_cloudfront_distribution.distribution E74FTE3EXAMPLE
```

네 이런식으로 AWS 서비스 하나씩 모듈화 TF 작성을 한 뒤에도 실제 생성되어 있는 리소스들을 yml 파일로 만들어서 한땀한땀 import 를 시켜주면 됩니다..

**테라폼 도입시도 세번째 - 모듈 분리 TF 장단점**

이런식으로 모듈을 분리하여 표준화 시키고 설정 yml 파일만 다르게 가져가면 사람인, 점핏, 플랩 등 서비스별로 테라폼을 사용하여 관리할 시 거의 동일한 구조를 가진 테라폼 TF 파일로 구성되어 있으므로 팀내 유지보수가 매우 용이합니다. 

실제로 현재 운영환경에서도 git 을 통해 소스 관리를 하여 작업 이력관리도 가능하고 다른 서비스의 리소스 작업이 필요할 경우에도 동일한 구조에서 yml 파일 일부를 수정해서 apply 를 하면 작업 접근성면에서 효율적이구요.

다만 이 방식의 단점은 점핏이면 점핏 플랩이면 플랩 프로젝트별 모든 리소스를 하나의 tfstate 파일에서 관리를 하기 때문에 terraform plan 명령어를 한번 엔터를 누르는 순간 수많은 리소스를 하나씩 다 검증하는 과정을 거쳐야 하기 때문에 종속성 체크 등 정확성은 뛰어나지만 작업 속도면에서 느립니다.

이 부분은 tfstate 파일을 서로 참조 가능하거나, 단독 분리가 가능한 모듈별로 분리하여 관리하면 시간 단축이 가능할 것 같으나 현재의 디렉토리 트리 구조를 다시 변경해야 할 것으로 예상되어 이번 프로젝트 단계에서는 개선 포인트로 남아 있습니다.

**테라폼 도입 후기**

2022년 상반기에 테라폼을 도입하여 현재까지 운영 환경에서 사용하면서 현업 부서에서의 갑작스러운 많은 양의 작업 요청에도 당황하지 않고 이미 작성된 tf, yml 파일을 뒤적이며 리소스들을 추가, 수정, 삭제하고 있습니다. 

예전 웹콘솔에서의 클릭만으로 작업을 해야하는 범위에서는 여기 고치고 저기 확인하고 다시 여기서 수정하고 내용 추가하고 하다보면 작업 히스토리 관리 측면에서도 그렇고 정신이 없을 지경이지만, 보안그룹이나 ELB 컨디션 등 코드로 관리하면 더 효율적이고 생산성이 높아지는 등 업무 효율성이 확실히 개선되었습니다.

테라폼을 도입하고 지금처럼 잘 사용을 해도 웹콘솔에서 일괄적인 확인작업을 병행하며 업무를 수행해야 하지만 확실히 업무에 많은 도움이 되는 도구임에는 틀림이 없습니다.

테라폼은 선언적이고 구조적인 HCL 언어를 사용 해야해서 불편한 점도 있는 것이 사실이나 이렇게 프로바이더별로 리소스들을 코드화 시켜 관리를 하면 점점 더 확장되는 서비스 이력 관리에도 용이하고 급하게 리소스를 추가하거나 DR 처리를 해야하는 중대한 상황에서도 리소스를 ‘백업’ 해 두었다는 효과가 있어서 확장 가능성을 확보할 수 있습니다. 

아직 테라폼 도입을 고려하고 계시는 분들이 계시다면 초기에 어려움이 있더라도 IaC 도구를 활용하여 웹콘솔 클릭 노가다로 부터 해방되면 참 좋다는 말씀을 드리면서 이 글을 마칩니다.

사람인 IT연구소 SRE팀 안재현이었습니다.

감사합니다.
