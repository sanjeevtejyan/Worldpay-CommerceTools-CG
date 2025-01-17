# Worldpay-commercetools Connector - Deployment Guide

This guide details how to deploy the connector within a production environment.

## Bootstrapping commercetools entities

To configure commercetools to invoke the connector extension endpoint, the API extensions need to be configured within your commercetools sandbox. Additionally, there are a number of resource type extensionsspecific to the Worldpay payment types.

| Name                               | commercetools extension type | Description                                                                                         |
| ---------------------------------- | ---------------------------- | --------------------------------------------------------------------------------------------------- |
| Payment Extension                  | API Extension                | API extension for the create/update payment actions                                                 |
| Worldpay Payment Type              | Custom Type                  | Payment resource custom fields for Worldpay specific data                                           |
| Payment Interface Interaction      | Custom Type                  | Worldpay payment interface interaction data (create payment request from commercetools to Worldpay) |
| Notification Interface Interaction | Custom Type                  | Worldpay notification interface interaction data (callback from Worldpay)                           |

These types are defined in the `packages/connector/resources/commercetools` folder. The relevant commercetools APIs need to be invoked to deploy these custom extensions into your commercetools workspace. To facilitate this, the connector provides bootstrapping logic that should be invoked once on any new commercetools sandbox utilising this connector.

To invoke the bootstrapping logic, ensure you have the required environment variables configured (see [user guide](USER_GUIDE.md) for details) and invoke the command `npm run bootstrap` - this will create each of the required entities within commercetools. _NB: This operation is idempotent and if the entities already exist, they will be ignored._

## Extension & Notification Endpoints

The Worldpay-Comercetools connector provides 2 x endpoints that need to be integrated within a commercetools environment.

- An API extension HTTP endpoint that is invoked by the commercetools platform whenever a create/update API call is made for a Payment resource. This is secured via HTTPS and HTTP authentication with a shared token configured within the connector and the bootstrapped API extension configuration.
- An API Notification endpoint that is invoked by the Worldpay Payment Gateway to send asynchronous updated regarding payment status changes. The connector processes these notification messages, and updates the relevant entities within commercetools. This endpoint is secured via HTTPS with Mutual TLS Certificate Authentication (mTLS)

## Deployment options

The module consists of a single stateless node.js application that supports both of these endpoints. It can be configured to respond to one or both of these endpoint calls.

Within a production environment, the connector can be deployed as a serverless function (AWS Lambda / Azure Function / GCP Cloud Function), with an API Gateway to invoke the function. Alternatively, it can be deployed as a docker container and run within your container management tooling of choice.

Reference infrastucture-as-code is provided for an AWS deployment using the [AWS Cloud Development Kit (CDK)](https://docs.aws.amazon.com/cdk/latest/guide/home.html) framework. This contains sample stacks for both a serverless (AWS Lambda) deployment or a containerised Docker deployment, using Fargate/ECS container management. _NB: You only need to choose and deploy one of these options - both are provided as reference_

The following diagrams shows the reference infrastructure for an AWS serverless deployment of the Worldpay-Commercetools connector module.

![AWS Serverless Infrastrcture](images/aws_serverless_infrastucture.png)

### AWS Serverless Infrastructure CDK Constructs

All of the AWS infrastucture components are available in the `packages/infra-aws` folder.

The following 2 steps enable you to deploy the Worldpay Commercetools Connector

#### 1. AWS CDK Configuration

Before deploying the stack, update the configuration values within the `cdk.json` file. Many of the details of the AWS environment can be modified such as memorySize and timeout for the function - these parameters are self-evident from the `cdk.json` file.
The key properties that need to be configured are as follows:

- `domain` - the domain that the API will be deployed to - i.e. _mycompany.com_
- `extensionSubdomain` - the subdomain that the commercetools API extension will interact with - i.e. _api_ - this will be prepended to the `domain` property when configuring the API Gateway - i.e. `api.mycompany.com`
- `notificationSubdomain` - the subdomain that the Worldpay WPG notifications will interact with - i.e. _api-secure_ - this will be secured with Mutual TLS Certification Authentication using the Worldpay root certificates, and hence must be a different subdomain as the [AWS Mutual TLS configuration](https://docs.aws.amazon.com/apigateway/latest/developerguide/rest-api-mutual-tls.html) is made against a REST API custom domain name.
- `stage` - the CDK stage - i.e. dev/tst/prd - the stage declared will be appended to the subdomain for all stages other than `prd` - i.e. `stage=dev` would create the API at `api-dev.mycompany.com`
- `certificateId` - ID of an SSL certificate that has been created within [AWS Certificate Manager](https://aws.amazon.com/certificate-manager/) for the `domain`
- `secretNames` - comma-separated list of one or more secrets defined within [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/) containing the commercetools and Worldpay credentials. These will be combined within any environment variables defined within Lambda/Docker config and made available to the node.js runtime as env vars. See the [user guide](USER_GUIDE.md) for details of the values that must be configured.

#### 2. Synthesize and deploy the stack

The `packages/infra-aws/lib` folder contains a number of TypeScript CDK constructs and stacks for all of the required infrastructure components, including Lambda function, IAM roles, API Gateway REST API and custom domain, Route53 DNS hosted zones, CloudWatch logs, S3 bucket for Worldpay mTLS certificate.

3 x stacks are available:

- `worldpay-certificate` - contains an S3 bucket with a deployment of the [Worldpay mTLS root certificate](https://developer.worldpay.com/docs/wpg/manage#authenticating-the-sender)
- `worldpay-connector-lambda` - stack containing all AWS infrastructure required for a serverless scalable Lambda deployment of the connector. This stack has a dependency on the `worldpay-certificate` stack.
- `worldpay-connector-docker` - stack containing all AWS infrastructure required for an ECS/Fargate deployment of the connector. This stack has a dependency on the `worldpay-certificate` stack.

The stack names will have the `stage` appended i.e. `worldpay-connector-lambda-dev`

You can choose whichever of the lambda or docker stacks suits your infrastruture guidelines.

To deploy the AWS CloudFormation stack for the `dev` stage;

- `npm run deploy -- worldpay-connector-lambda-dev --context stage=dev --context account=1234567890`

  or

- `npm run deploy -- worldpay-connector-docker-dev --context stage=dev --context account=1234567890`
