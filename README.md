# api-gateway-test

This is a sample template for api-gateway-test - Below is a brief explanation of what we have generated for you:

```bash
.
├── README.MD                   <-- This instructions file
├── src                         <-- Source code for a lambda function
│   └── app.js                  <-- Lambda function code
│   └── package.json            <-- NodeJS dependencies and scripts
│   └── tests                   <-- Unit tests
│       └── unit
│           └── test-handler.js
├── template.yaml               <-- SAM template
├── swagger.yaml                <-- Swagger definition
├── buildspec.yml               <-- Build file processed by pipeline
```

## Requirements

* AWS CLI already configured with Administrator permission
* [NodeJS 10.10+ installed](https://nodejs.org/en/download/releases/)
* [Docker installed](https://www.docker.com/community-edition)

## Setup process

### Local development

**Invoking function locally using a local sample payload**

```bash
sam local invoke GetUsersFunction --no-event
```
 [Local invoke parameters](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-cli-command-reference-sam-local-invoke.html)
 
**Invoking function locally through local API Gateway**

```bash
sam local start-api
```

If the previous command ran successfully you should now be able to hit the following local endpoint to invoke your function `http://localhost:3000/users`

**SAM CLI** is used to emulate both Lambda and API Gateway locally and uses our `template.yaml` to understand how to bootstrap this environment (runtime, where the source code is, etc.) - The following excerpt is what the CLI will read in order to initialize an API generate resources/methods from the swagger file.

```yaml
...
  ServerlessRestAPI:
    Type: AWS::Serverless::Api
    Properties:
      StageName: dev
      EndpointConfiguration: REGIONAL
      DefinitionBody:
        'Fn::Transform':
          Name: 'AWS::Include'
          Parameters:
            Location: './swagger.yaml'
```

The swagger file also contains where Lambda invocations will be called on each method/resource (GetUsersFunction maps to the function created in our template):

```yaml
...
paths:
  /users:
    get:
      description: Gets full list of users
      produces:
        - application/json
      responses:
        '200':
          description: The request was successful.
          schema:
            type: array
            items:
              $ref: '#/definitions/User'
      x-amazon-apigateway-integration:
        uri: 
            Fn::Sub: arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${GetUsersFunction.Arn}/invocations
        passthroughBehavior: when_no_match
        httpMethod: POST
        type: aws_proxy
        ...
```

## Packaging and deployment

AWS Lambda NodeJS runtime requires a flat folder with all dependencies including the application. SAM will use `CodeUri` property to know where to look up for both application and dependencies:

```yaml
...
    GetUsersFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      ...
```

Firstly, we need a `S3 bucket` where we can upload our Lambda functions packaged as ZIP before we deploy anything - If you don't have a S3 bucket to store code artifacts then this is a good time to create one:

```bash
aws s3 mb s3://BUCKET_NAME
```
As long as the pipeline has been built correctly using the Terraform a simple git commit and push to the correct repo/branch should trigger the code being built/deployed to the API Gateway.

> **See [Serverless Application Model (SAM) HOWTO Guide](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-quick-start.html) for more details in how to get started.**

You can find more information and examples about filtering Lambda function logs in the [SAM CLI Documentation](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-logging.html).

## Testing

We use `mocha` for testing our code and it is already added in `package.json` under `scripts`, so that we can simply run the following command to run our tests:

```bash
cd src
npm install
npm run test
```

## Cleanup

In order to delete our Serverless Application recently deployed you just need to delete the stack from cloud formation.

## Step-through debugging

* **[Enable step-through debugging docs for supported runtimes]((https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-using-debugging.html))**

Next, you can use AWS Serverless Application Repository to deploy ready to use Apps that go beyond hello world samples and learn how authors developed their applications: [AWS Serverless Application Repository main page](https://aws.amazon.com/serverless/serverlessrepo/)

# Appendix

## Building the project

[AWS Lambda requires a flat folder](https://docs.aws.amazon.com/lambda/latest/dg/nodejs-create-deployment-pkg.html) with the application as well as its dependencies in a node_modules folder. When you make changes to your source code or dependency manifest, run the following command to build your project local testing and deployment:

```bash
sam build
```

If your dependencies contain native modules that need to be compiled specifically for the operating system running on AWS Lambda, use this command to build inside a Lambda-like Docker container instead:
```bash
sam build --use-container
```

By default, this command writes built artifacts to `.aws-sam/build` folder.
