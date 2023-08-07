---
layout: post
title: Integrated AWS REST API Gateway with AWS Lambda without resource policy
date: 2022-12-05 15:00:00 +0100
categories: learning
tags: aws, apigw, terraform, lambda
---

## Overview

When I looked into the Terraform documentation. I found that the documentation
is using the resource policy to allow API Gateway to invoke the lambda function
by using the `aws_lambda_permission`[^1] resource to allow various resources
to invoke the lambda.

This piqued my interest that is it possible to use IAM role to invoke the
lambda function instead of using the resource policy in the lambda function.

This post will show you how to allow REST API Gateway to invoke the
lambda function via IAM Role instead of resource policy.

<!-- markdownlint-disable-next-line MD001 -->
> #### DANGER
>
> Please don't just copy paste this example because these configurations
> is for presentation purposes. There's no security hardening in place.
> For example, there's no authentication and authorization when the end-user
> call to API Gateway which make your API expose to the internet without
> any authentication.
>
> Please consider following [OWASP API Security project](https://owasp.org/www-project-api-security/)
> before deploy the API to the public internet.
{: .block-danger}

## Prerequisite

We expect the reader to have the basic understanding of:

- Terraform[^2] and aws provider[^3]
- AWS Rest API Gateway[^4]
- AWS IAM[^5]
- AWS Lambda[^6]

## Steps

In this section, we will focus on the IAM and API Gateway configuration.
As mentioned earlier, this documentation will use Terraform as a configuration
tools.

<!-- markdownlint-disable-next-line MD001 -->
> #### WARNING
>
> There are various way to configure API Gateway. We will use Terraform
> to configure the integration. Also, we will show only the changes terraform
> resources. If you want to know how to configure everything
> please follow the Terraform AWS provider for REST API Gateway documentation[^7].
{: .block-warning}

### Create IAM Role for API Gateway

There are various way to create the IAM role and policy. In this post,
we will try to use pure Terraform as much as possible.

First, we will define the data source for API Gateway to assume.

```terraform
data "aws_iam_policy_document" "apigateway_assume_role" {
  statement {
    actions = [
      "sts:AssumeRole"
    ]

    principals {
      type = "Service"
      identifiers = [
        "apigateway.amazonaws.com"
      ]
    }
  }
}
```

Next, we will specify the data source to give API Gateway permission
to invoke lambda function.

```terraform
data "aws_iam_policy_document" "apigateway_role" {
  statement {
    actions = [
      "lambda:InvokeFunction"
    ]

    resources = [
      aws_lambda_function.winter_app_lambda.arn # you can specify arn with wildcard to allow invoking every lambda function
    ]
  }
}
```

Finally, create the IAM role and policy then attach to the created role.

```terraform
resource "aws_iam_role" "apigw_role" {
  name               = "api_gateway_execution_role"
  assume_role_policy = data.aws_iam_policy_document.apigateway_assume_role.json
}

resource "aws_iam_policy" "winter_app_apigw_policy" {
  name = "api_gateway_execution_policy"
  policy = data.aws_iam_policy_document.apigateway_role.json
}

resource "aws_iam_role_policy_attachment" "apigw_role" {
  role = aws_iam_role.apigw_role.name
  policy_arn = aws_iam_policy.apigw_policy.arn
}
```

Now, we have the IAM role ready for the API Gateway to assume
and invoke the lambda function.

### Configure API Gateway to assume the role

In Terraform, we will focusing on `aws_api_gateway_method` and `aws_api_gateway_integration`
resources because these resources will determine how we integrated with
the lambda function and how to setup the authorization.

In `aws_api_gateway_method`, instead of using `None` in authorization
argument, we will use `AWS_IAM` to specify that API Gateway need to use IAM role
to invoke the lambda.

```terraform
resource "aws_api_gateway_method" "winter_app_lambda" {
  rest_api_id      = resource.aws_api_gateway_rest_api.api_gateway.id
  resource_id      = resource.aws_api_gateway_resource.lambda_function.id
  http_method      = "ANY"
  authorization    = "AWS_IAM" # <- change this line
  api_key_required = false
}
```

However, to specify the role for API Gateway to assume, we need to configure
it in `aws_api_gateway_integration` resource by adding `credentials` argument.

```terraform
resource "aws_api_gateway_integration" "winter_app_lambda" {
  rest_api_id             = aws_api_gateway_rest_api.api_gateway.id
  resource_id             = aws_api_gateway_resource.lambda_function.id
  http_method             = aws_api_gateway_method.lambda_function.http_method
  integration_http_method = "ANY"
  type                    = "AWS_PROXY"
  uri                     = aws_lambda_function.lambda_function.invoke_arn
  credentials             = aws_iam_role.apigw_role.arn # <- adding this line
}
```

## Conclusion

That's it folks! we now having the API Gateway that can invoke the lambda
function with IAM Role.

[^1]: [Specify Lambda Permissions for API Gateway REST API](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lambda_permission#specify-lambda-permissions-for-api-gateway-rest-api)
[^2]: [Terraform](https://developer.hashicorp.com/terraform/docs)
[^3]: [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
[^4]: [Amazon API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html)
[^5]: [Amazon IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html)
[^6]: [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/operatorguide/intro.html)
[^7]: [Use Terraform resources to configure REST API Gateway](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/api_gateway_rest_api#terraform-resources)
