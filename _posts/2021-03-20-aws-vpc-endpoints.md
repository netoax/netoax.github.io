---
layout: post
title: "Enabling ECS Fargate 1.4.0 to pull images from ECR"
author: "João Neto"
categories: cloud
tags: [aws, terraform, ecr, fargate, ecs, vpc endpoint, cloud computing]
---

In the last year, AWS has launched the [Fargate](https://aws.amazon.com/blogs/containers/aws-fargate-launches-platform-version-1-4/) platform version `1.4.0`. It comes with nice features such as Elastic File System (EFS) endpoints and increased ephemeral volumes (20GB). Also, it changed the task Elastic Network Interface (ENI) to control additional networking traffic flows, which were controlled by the Fargate ENIs in older versions. These networking flows include login with Elastic Container Registry (ECR) and sourcing secrets from Secrets Manager.

From the customer perspective, the main drawback of using Fargate ENIs for this purpose is the lack of visibility and control since they're managed by AWS. On the contrary, the Task ENIs use the same networking configuration from the customer VPC.

Recently, I've updated the ECS services of my project to use the Fargate `1.4.0` version without paying attention to the change in ECR login networking flow. So, in this post, I will summarize the changes I did to successfully finish the upgrade.

## AWS PrivateLink and VPC Endpoints

The AWS PrivateLink is a solution that enables private connection between the customer VPC and other services managed by AWS without exposing your services to the internet. It helps to save $$ by avoiding the need of configuring Internet Gateway, NAT or VPN.

In this context, the [VPC Endpoint](https://docs.aws.amazon.com/vpc/latest/privatelink/endpoint-services-overview.html) is an entry point on VPC-side to enable that private connection. It comes in three different types:

- **Interface**: allows connection with AWS services (ECR, Secrets Manager, etc).
- **Gateway**: gateway configured in the routing table to target a required AWS service.
- **Gateway Load Balancer**: intercept traffic and route it to a custom service configured through Gateway Load Balancers.

The following VPC Endpoints must be created so that ECS Fargate can pull images from the ECR:

- **Interface Endpoint for ECR service (download)** (`com.amazonaws.<region>.ecr.dkr`): allows to pull the image manifests from the ECR.
- **Gateway Endpoint for S3 service** (`com.amazonaws.<region>.s3`): allows the services to download image layers from the private S3 buckets that store them.

As I said before, in version `1.4.0` the login to ECR services is now controlled by the Task ENIs. Therefore, it's now necessary to setup a new VPC Endpoint:

- **Interface Endpoint for ECR service (login)** (`com.amazonaws.<region>.ecr.api`): allows login to the ECR.

## Terraform Example

This section assumes you have a configured VPC (in this example we're using a [module](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest)).

### Security Group

Firstly, create a security group to protect your VPC Endpoints (only for `Interface` type). It should have an **inbound** rule allowing HTTPS requests on `443` port from the ECS services.

```hcl
resource "aws_security_group" "vpce" {
  name        = "example-vpce-sg"
  description = "Security group to control VPC Endpoints inbound/outbound rules"
  vpc_id      = module.vpc.vpc_id
  depends_on  = [module.vpc]

  ingress {
    from_port       = 443
    to_port         = 443
    protocol        = "tcp"
    security_groups = [aws_security_group.ecs_service.id]
  }

  tags = {
    Name = "vpce-sg"
  }
}
```

In the ECS services security group, you should allow the same HTTPS traffic as an **outbound** rule with `prefix_list_id` output from S3 Endpoint.

```hcl
resource "aws_security_group" "ecs_service" {
  vpc_id      = module.vpc.vpc_id
  name        = "ecs-service-sg"
  description = "Security group to control ECS services inbound/outbound rules"

  egress {
    from_port       = 443
    to_port         = 443
    protocol        = "tcp"
    prefix_list_ids = [aws_vpc_endpoint.s3.prefix_list_id]
  }
}
```

### Gateway Endpoint for S3


```hcl
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = module.vpc.vpc_id
  service_name      = "com.amazonaws.${var.region}.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = module.vpc.private_route_table_ids

  tags = {
    Name = "vpce-s3"
  }
}
```

### Interface Endpoints for ECR

The following endpoint allows to download the image manifest from the ECR service.

```hcl
resource "aws_vpc_endpoint" "dkr" {
  vpc_id              = module.vpc.vpc_id
  private_dns_enabled = true
  service_name        = "com.amazonaws.${var.region}.ecr.dkr"
  vpc_endpoint_type   = "Interface"
  security_group_ids  = [aws_security_group.vpce.id]
  subnet_ids          = module.vpc.private_subnets

  tags = {
    Name = "vpce-dkr"
  }
```

As explained [here](#aws-privatelink-and-vpc-endpoints), the `ecr.api` endpoint is now required to allow the services to login to the ECR services. It can be done in the following way:


```hcl
resource "aws_vpc_endpoint" "ecr" {
  vpc_id              = module.vpc.vpc_id
  private_dns_enabled = true
  service_name        = "com.amazonaws.${var.region}.ecr.api"
  vpc_endpoint_type   = "Interface"
  security_group_ids  = [aws_security_group.vpce.id]
  subnet_ids          = module.vpc.private_subnets

  tags = {
    Name = "vpce-ecr-api"
  }
}
```