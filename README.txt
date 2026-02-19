h1. Centralized API Gateway Ownership Model for AWS Cloud

Author: Enterprise Integration Platform (EIP) Team
Purpose: Justification for centralized ownership and governance of AWS API Gateway across multi-account architecture

h2. Executive Summary

As the organization migrates from WSO2 Gateway (on-prem) to AWS API Gateway (cloud), it is critical that the Enterprise Integration Platform (EIP) team maintains centralized ownership of API Gateway infrastructure.

While API Gateway instances will exist in each AWS account for isolation and scalability, configuration and governance must remain centrally controlled by the platform team.

API Gateway is not an application component — it is part of the enterprise security and traffic control plane.

Application teams should own business logic, but not authentication enforcement, routing policies, or gateway security.

h2. Architectural Principle: Control Plane vs Data Plane

Enterprise cloud architectures separate responsibilities into two layers.

h3. Control Plane (Platform Team Owned)

Responsible for security, routing, and governance:

AWS API Gateway

Lambda Authorizer

IAM policies controlling API access

JWT validation logic

Throttling policies

Routing enforcement

WAF integration

Terraform infrastructure modules

h3. Data Plane (Application Team Owned)

Responsible for business logic:

Lambda functions

Step Functions workflows

DynamoDB / RDS / S3

Business logic

OpenAPI contract definition

This separation prevents security vulnerabilities and operational instability.

h2. Why Application Teams Must NOT Own API Gateway

h3. 1. Authentication Enforcement is a Security Boundary

If application teams control API Gateway, they can accidentally deploy insecure configurations.

Example mistake:

{code:language=terraform}
resource "aws_apigatewayv2_route" "insecure_route" {
api_id = aws_apigatewayv2_api.api.id
route_key = "GET /customer"

authorization_type = "NONE"
}
{code}

This exposes internal APIs publicly.

Correct configuration enforced by platform team:

{code:language=terraform}
resource "aws_apigatewayv2_route" "secure_route" {
api_id = aws_apigatewayv2_api.api.id
route_key = "GET /customer"

authorization_type = "CUSTOM"
authorizer_id = aws_apigatewayv2_authorizer.jwt_authorizer.id
}
{code}

h3. 2. JWT Validation Must Be Uniform Across Enterprise

Incorrect validation allows unauthorized access.

Application team mistake example:

{code:language=terraform}
resource "aws_apigatewayv2_authorizer" "bad_authorizer" {
api_id = aws_apigatewayv2_api.api.id
authorizer_type = "JWT"

identity_sources = ["$request.header.Authorization"]

jwt_configuration {
issuer = "https://wrong-issuer.com
"
audience = ["any"]
}
}
{code}

Platform team ensures strict validation:

{code:language=terraform}
resource "aws_apigatewayv2_authorizer" "secure_authorizer" {
api_id = aws_apigatewayv2_api.api.id
authorizer_type = "JWT"

identity_sources = ["$request.header.Authorization"]

jwt_configuration {
issuer = "https://login.microsoftonline.com/
<tenant>/v2.0"
audience = ["enterprise-api"]
}
}
{code}

h3. 3. Application Teams May Bypass Gateway Entirely

Critical security risk:

{code:language=terraform}
resource "aws_lambda_function_url" "dangerous" {
function_name = aws_lambda_function.backend.function_name
authorization_type = "NONE"
}
{code}

This exposes backend directly to internet.

Correct approach enforced by platform team:

{code:language=terraform}
authorization_type = "AWS_IAM"
{code}

Or backend accessible only through API Gateway integration.

h3. 4. IAM Policy Misconfiguration Creates Privilege Escalation Risk

Application team mistake:

{code:language=terraform}
resource "aws_iam_role_policy" "overprivileged" {
role = aws_iam_role.lambda_role.id

policy = jsonencode({
Statement = [{
Effect = "Allow"
Action = ""
Resource = ""
}]
})
}
{code}

Platform team enforces least privilege:

{code:language=terraform}
Statement = [{
Effect = "Allow"
Action = [
"dynamodb:GetItem"
]
Resource = aws_dynamodb_table.customer.arn
}]
{code}

h3. 5. Routing Conflicts in Shared Account

Multiple teams modifying gateway can create conflicts.

Team A deploys:

{code:language=terraform}
route_key = "GET /customer"
{code}

Team B deploys same route:

{code:language=terraform}
route_key = "GET /customer"
{code}

Production traffic breaks.

Platform team manages routing registry and prevents conflicts.

h3. 6. Throttling and DoS Protection Misconfiguration

Application team mistake:

{code:language=terraform}
resource "aws_apigatewayv2_stage" "prod" {
default_route_settings {
throttling_rate_limit = 100000
}
}
{code}

Platform team enforces safe throttling limits.

h3. 7. Logging and Observability Gaps

Application team mistake:

{code:language=terraform}
access_log_settings {
destination_arn = null
}
{code}

Platform team enforced logging:

{code:language=terraform}
access_log_settings {
destination_arn = aws_cloudwatch_log_group.apigw.arn
}
{code}

h2. Platform-Owned Terraform Gateway Module (Correct Model)

Platform team provides reusable module:

{code:language=terraform}
module "api_gateway" {
source = "git::ssh://git.enterprise/api-platform-modules/apigateway"

api_name = var.api_name
jwt_issuer = var.jwt_issuer
jwt_audience = var.jwt_audience
enable_waf = true
enable_logging = true
enable_throttle = true
}
{code}

Application teams provide only OpenAPI spec.

Platform team controls security implementation.

h2. Cross-Account Deployment Enforcement

Secure deployment role:

{code:language=terraform}
resource "aws_iam_role" "api_platform_deploy_role" {
assume_role_policy = jsonencode({
Statement = [{
Effect = "Allow"
Principal = {
Federated = "arn:aws:iam::<account>:oidc-provider/token.actions.githubusercontent.com"
}
Action = "sts:AssumeRoleWithWebIdentity"
}]
})
}
{code}

Application teams cannot bypass governance.

h2. Compliance and Audit Requirements

Central ownership ensures compliance with:

SOC2

PCI DSS

SOX

FFIEC

Platform team ensures:

CloudTrail enabled

Consistent authentication

Audit logging

h2. Recommended Ownership Model

h3. Platform Team Owns

API Gateway

Lambda Authorizer

IAM policies

Terraform modules

Authentication enforcement

Routing policies

Gateway deployment pipeline

h3. Application Teams Own

Business Lambda

Step Functions

Database

OpenAPI contract

h2. Risk Comparison

|| Ownership Model || Security Risk || Compliance Risk || Operational Risk ||
| Platform Owned | Low | Low | Low |
| Application Owned | High | High | High |

h2. Final Recommendation

The Enterprise Integration Platform team must maintain centralized ownership of AWS API Gateway.

Application teams must not directly configure or deploy API Gateway infrastructure.

This ensures:

Security consistency

Compliance enforcement

Operational stability

Proper governance

Enterprise scalability

This model aligns with enterprise cloud architecture and industry best practices.
