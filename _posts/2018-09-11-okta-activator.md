---
layout: post
title: "Empowering Okta Helpdesk Admins to Activate Users"
date: 2018-09-11
---

## Introduction - The Problem

At Unbounce we have a remote office in Berlin that we want to keep empowered and relatively autonomous from our HQ office in Vancouver, BC., but we don't have an I.T. team on the ground there. To that end we have granted basic help desk admin priveleges to most platforms including Okta. At the time my thinking for granting the Okta Help Desk role was so that new employees in Berlin could be activated when they arrived for onboarding by a member of the team there. Unfortunately I quickly discovered that Help Desk admins are not permitted to Activate users in Okta. Hmmmm. Without the ability to activate users it means our I.T. team back in Vancouver have to either activate the user the previous day or get up in the middle of the night to do it. Neither of these are ideal. 

Okta has an API that allows you to activate users - so initially I considered creating CRON jobs every time we expected a new hire in our Berlin office, but was kind of offput by the idea of having to maintain these jobs, especially if we ramped up hiring. After thinking through it for a little bit longer I decided what I wanted to do was to provide the ability for one of our Help Desk admins in Berlin to trigger this Okta API endpoint, but without exposing them to any further functionality or the API key of the Super Admin used to make the call. 

## The Solution

The solution I settled on works on the following basic principle - the API call to Okta is made by AWS Lambda (in an AWS account only accessible by the I.T. Team and Security), the Lambda function is triggered by a custom API implemented in AWS API Gateway, and the end user triggers our custom API endpoint using a custom Go application. 

Lambda Implementation
=====================

The Lambda Python code here is pretty straight forward. Written in Python3.6 it uses the `requests` module to make a very standard API call, and handle any errors that can potentially be returned by the Okta api. We'll step through each function before looking at the entire code towards the end of this section. 

```python
def get_api_key():
    ssm = boto3.client('ssm', region_name=SESSION.region_name)
    response = ssm.get_parameter(
        Name=SSM_OBJECT,
        WithDecryption=True)
    credentials = response['Parameter']['Value']
    return credentials
```
The first thing we need to do is give our Lambda function access to the Okta API Token that we will use to make the call. It's generally bad practice to store secrets as environment variables in Lambda as they can be exposed over the API, so we are going to use the above function to request access to AWS SSM where we will later store the API key.

```python
def get_user_id(user_login):
    url = f"{BASE_URL}users/{user_login}"
    r = requests.get(url, headers=HEADERS)
    if r.status_code == 404:
        print(r.status_code)
        raise Exception("404")
    else:
        user_data = r.json()
        print(r.status_code)
        return user_data["id"]
```
The Okta API expects a user ID as the identifying parameter when activating a user, this isn't particularly useful for an end user who doesn't have access to that information. Thankfully we can use the `Get User` endpoint to find a user by their login (email in our case). Once we've retrieved the user we can pass their `id` value to our activation function. In this function we also need to raise an error if the Okta API returns a 404, indicating the requested user does not exist in our organisation. It's important to raise these as errors instead of returning just status codes, as the API gateway will always return a 200 OK unless the Lambda function raised an error, regardless of the returned message body. 

Now that we know the unique `id` for the user we want to activate we can send that to our activation function. 

```python
def activate_user(user_id):
    url = f"{BASE_URL}users/{user_id}/lifecycle/activate"
    r = requests.post(url, headers=HEADERS)
    status = r.status_code
    print(r.json())
    return(status)
```

This one's pretty straight foward - it simply calls out to the User Activation endpoint of the Okta API. 

Putting it all together the full script looks like:

```python
import os

import boto3
from botocore.vendored import requests

BASE_URL = os.getenv("OKTA_URL") # e.g. https://your-company.okta.com/api/v1/
SESSION = boto3.session.Session() # so we can retrieve information about where Lambda is running
SSM_OBJECT = os.getenv("SSM_OBJECT") # the name of the SSM parameter where the Okta API key is stored

HEADERS = {
    "Accept": "application/json",
    "Content-Type": "application/json",
    "Authorization": "SSWS {}"
}

def get_api_key():
    ssm = boto3.client('ssm', region_name=SESSION.region_name)
    response = ssm.get_parameter(
        Name=SSM_OBJECT,
        WithDecryption=True)
    credentials = response['Parameter']['Value']
    return credentials

def get_user_id(user_login):
    url = f"{BASE_URL}users/{user_login}"
    r = requests.get(url, headers=HEADERS)
    if r.status_code == 404:
        print(r.status_code)
        raise Exception("404")
    else:
        user_data = r.json()
        print(r.status_code)
        return user_data["id"]


def activate_user(user_id):
    url = f"{BASE_URL}users/{user_id}/lifecycle/activate"
    r = requests.post(url, headers=HEADERS)
    status = r.status_code
    print(r.json())
    return(status)


def handler(event, context):
    print(event)
    token=get_api_key()
    HEADERS=HEADERS.format(token)
    user_id = get_user_id(event["username"])
    status = activate_user(user_id)
    print(status)
    if status == 200:
        return {
            "code": "200",
            "message": f"{event['username']} successfully activated."
        }
    else:
        raise Exception("403")
```

You could manually create this function via the AWS Console but in my case I prefer to have all of my AWS services deployed via Cloudformation or CLI so that the infrastructure can be codified and easily reviewed/audited. Here is the excerpt of my Cloudformation script that create's the Lambda function from a packaged version of the above code hosted in S3 (learn more about packaging Lambda code [here][1]). 

```yaml
LambdaFunction:
        Type: "AWS::Lambda::Function"
        Properties:
            FunctionName: "okta-activate"
            Handler: "okta-activate.handler"
            MemorySize: 128
            Role: !GetAtt ["LambdaExecutionRole", "Arn"]
            Runtime: "python3.6"
            Timeout: 3
            Code: 
                S3Bucket: !ImportValue "lambda-s3-bucket:bucket-name"
                S3Key: !Ref S3Key
            Environment:
                Variables:
                    OKTA_URL: !Ref OktaUrl
                    SSM_OBJECT: !Ref SSMObjectEnv
            
    LambdaExecutionRole:
        Type: "AWS::IAM::Role"
        Properties: 
            AssumeRolePolicyDocument:
                Version: '2012-10-17'
                Statement:
                - Effect: Allow
                  Principal: 
                    Service: lambda.amazonaws.com
                  Action:
                  - sts:AssumeRole
            Path: '/'
            Policies:
            - PolicyName: GetSSMValue
              PolicyDocument:
                Version: "2012-10-17"
                Statement:
                - Effect: Allow
                  Action:
                  -  "ssm:DescribeParameters"
                  -  "ssm:GetParameter*"
                  Resource: !Sub "arn:aws:ssm:${AWS::Region}:${AWS::AccountId}:parameter${SSMObjectEnv}"
            - PolicyName: DecryptSSMValue
              PolicyDocument:
                Version: '2012-10-17'
                Statement:
                - Effect: Allow
                  Action:
                  - "kms:Decrypt"
                  Resource: !GetAtt KMSKey.Arn
            - PolicyName: "allow-lambda-logging"
              PolicyDocument:
                Version: '2012-10-17'
                Statement:
                    -   Effect: Allow
                        Action:
                            - "logs:CreateLogGroup"
                            - "logs:CreateLogStream"
                            - "logs:PutLogEvents"
                        Resource: !Sub "arn:aws:logs:${AWS::Region}:${AWS::AccountId}:*"

    KMSKey:
        Type: "AWS::KMS::Key"
        Properties:
            Description: "Key to decrypt Okta API token parameter"
            Enabled: true
            KeyPolicy:
                Version: "2012-10-17"
                Id: "okta-api-kms-key"
                Statement:
                    -
                        Sid: "Allow administration of the key"
                        Effect: "Allow"
                        Principal: 
                            AWS: !Sub "arn:aws:iam::${AWS::AccountId}:root"
                        Action: 
                            - "kms:Create*"
                            - "kms:Describe*"
                            - "kms:Enable*"
                            - "kms:List*"
                            - "kms:Put*"
                            - "kms:Update*"
                            - "kms:Revoke*"
                            - "kms:Disable*"
                            - "kms:Get*"
                            - "kms:Delete*"
                            - "kms:ScheduleKeyDeletion"
                            - "kms:CancelKeyDeletion"
                        Resource: "*"
```

In my case - I want to keep my Okta API token encrypted in SSM and ideally I'd do that by creating the SSM `SecureString` object and KMS key all in my Cloudformation template, but unfortunately Cloudformation doesn't currently support creating `SecureString` type objects for SSM. As a result - I'm only using Cloudformation to create the KMS key, and once the stack is live I can use the AWS CLI tool to create my encrypted parameter with the following simple bash script:

```bash
#!/bin/bash

# This must be run after the CFN Stack has been deployed. Creates an SSM object to store the Okta API key in, encrypted by 
# KMS key from CFN stack. 

API_KEY=$1
PROFILE=$2
KEY_ID=$3
REGION=$4

aws ssm put-parameter --name 'SSM/object/name' --value $API_KEY --type 'SecureString' --key-id $KEY_ID --profile $PROFILE --region $REGION
```

To summarise what's going on here - we first create a function in Lambda using the `AWS::Lambda::Function` Cloudformation type. We make sure to keep all sensitive and variable data out of our template - opting to pass parameters and create environment variables where needed. We also create the IAM role that the Lambda function will execute with - giving it permission to execute, retrieve and decrypt our SSM parameter, and access Cloudwatch for logging. We then create a KMS key that we can use to encrypt and decrypt the parameter we store in SSM. Outside of Cloudformation - we use the AWS CLI to create the SSM parameter as a `SecureString` and encrypt it using the key we created earlier. 

Now that our Lambda function is set up we can move onto the next step of creating the API Gateway.

API Gateway
===============

Creating an API endpoint in AWS API Gateway via Cloudformation can be...frustrating. It's a lot of satisfying conditions and elements for the sake of it, rather than because you know exactly what they do (especially in a non-product, small scale application like this). For simplicty though we can basically comprise an API service of the following elements:

* A `stage` - in a product environment this would be used to segregate production, development, integration, etc. from one another. In ours we aren't really using that approach so we just create one production stage. 
* A `deployment` - after creating or changing some configurations in API Gateway you need to explicitly deploy them before they are available to our end users. 
* A `method` - a REST API method. In our case we are using a standard `POST` method. This is where most of our actual configuration takes place
* A `usage plan` - this is required in order to associate an API key with an API. 
* An `API Key` - we don't want our API to be unauthenticated and open to the world, so we use an API key to provide some authentication. 

As noted, our `method` contains most of the logic and configuration we need to make this all work. Lets take a look at that section of our Cloudformation template. 

```yaml
ApiPost:
    Type: "AWS::ApiGateway::Method"
    Properties:
        AuthorizationType: None
        ApiKeyRequired: true
        HttpMethod: "POST"
        RestApiId: !Ref OktaActivateApi
        ResourceId: !GetAtt ["OktaActivateApi", "RootResourceId"]
        Integration: 
            Type: "AWS"
            IntegrationHttpMethod: "POST"
            Uri: !Sub arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${LambdaFunction.Arn}/invocations
            IntegrationResponses:
                -   StatusCode: 200
                    ResponseTemplates:
                        application/json: ""
                -   StatusCode: 403
                    SelectionPattern: ".*403.*"
                    ResponseTemplates:
                        application/json: "#set($inputRoot = $input.path('$'))\n{\n  \"message\"\
            \ : \"403 - User cannot be activated\"\n}"
                -   StatusCode: 404
                    SelectionPattern: ".*404.*"
                    ResponseTemplates:
                        application/json: "#set($inputRoot = $input.path('$'))\n{\n  \"message\"\
            \ : \"404 - User not found!\"\n}"
            PassthroughBehavior: WHEN_NO_TEMPLATES
        MethodResponses:
            -   StatusCode: 200
            -   StatusCode: 403
            -   StatusCode: 404
```
We're doing a couple of things here. At the top we're setting some high level attributes - making sure that we are requiring an API key to invoke the function, setting the method type (`POST`) etc. The bulk of what we're doing here though is error handling from our Lambda function back to our API method. We want to make sure that if the Okta API call fails in our Lambda function we can return useful information to the end user. This is why it was so important for us to raise errors in our Python code rather than returning content in the message body. The `SelectionPattern` attribute will only actually look for patterns in Lambda Error messages, it does not scan content returned from the handler. When the API see's a message returned from the Lambda error that matches the designated patterns (i.e. 403 or 404) it passes some JSON back to the method and tells the method what code to return with that message. 

Here's the full section of our Cloudformation template that deals with the API elements.

```yaml
OktaActivateApi:
        Type: "AWS::ApiGateway::RestApi"
        Properties:
            Name: "okta-activate-api"
            Description: "API Endpoint to trigger an Okta user activation action via Lambda"
    
OktaActivateApiStage:
    Type: "AWS::ApiGateway::Stage"
    Properties:
        DeploymentId: !Ref OktaActivateApiDeployment
        MethodSettings:
            -   DataTraceEnabled: true
                HttpMethod: "POST"
                LoggingLevel: "INFO"
                ResourcePath: "/*"
        RestApiId: !Ref OktaActivateApi
        StageName: !Ref OktaActivateApiDeployment

OktaActivateApiDeployment:
    Type: "AWS::ApiGateway::Deployment"
    DependsOn: ["ApiPost"]
    Properties:
        RestApiId: !Ref OktaActivateApi
        StageName: "prod"

ApiResource:
    Type: "AWS::ApiGateway::Resource"
    Properties:
        RestApiId: !Ref OktaActivateApi
        ParentId: !GetAtt ["OktaActivateApi", "RootResourceId"]
        PathPart: "okta-activate"

ApiPost:
    Type: "AWS::ApiGateway::Method"
    Properties:
        AuthorizationType: None
        ApiKeyRequired: true
        HttpMethod: "POST"
        RestApiId: !Ref OktaActivateApi
        ResourceId: !GetAtt ["OktaActivateApi", "RootResourceId"]
        Integration: 
            Type: "AWS"
            IntegrationHttpMethod: "POST"
            Uri: !Sub arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${LambdaFunction.Arn}/invocations
            IntegrationResponses:
                -   StatusCode: 200
                    ResponseTemplates:
                        application/json: ""
                -   StatusCode: 403
                    SelectionPattern: ".*403.*"
                    ResponseTemplates:
                        application/json: "#set($inputRoot = $input.path('$'))\n{\n  \"message\"\
            \ : \"403 - User cannot be activated\"\n}"
                -   StatusCode: 404
                    SelectionPattern: ".*404.*"
                    ResponseTemplates:
                        application/json: "#set($inputRoot = $input.path('$'))\n{\n  \"message\"\
            \ : \"404 - User not found!\"\n}"
            PassthroughBehavior: WHEN_NO_TEMPLATES
        MethodResponses:
            -   StatusCode: 200
            -   StatusCode: 403
            -   StatusCode: 404

ApiUsagePlan:
    Type: "AWS::ApiGateway::UsagePlan"
    Properties:
        ApiStages:
        -   ApiId: !Ref OktaActivateApi
            Stage: !Ref OktaActivateApiStage
        UsagePlanName: "okta-activate-usage-plan"
```

And below is the full Cloudformation template.

```yaml
---
AWSTemplateFormatVersion: "2010-09-09"
Description: "Deploy Lambda function and API Gateway for the okta-activate tool"

Parameters:
    OktaUrl:
        Type: String
    SSMObjectEnv:
        Type: String
    S3Key:
        Type: String

Resources:
    LambdaFunction:
        Type: "AWS::Lambda::Function"
        Properties:
            FunctionName: "okta-activate"
            Handler: "okta-activate.handler"
            MemorySize: 128
            Role: !GetAtt ["LambdaExecutionRole", "Arn"]
            Runtime: "python3.6"
            Timeout: 3
            Code: 
                S3Bucket: !ImportValue "lambda-s3-bucket:bucket-name"
                S3Key: !Ref S3Key
            Environment:
                Variables:
                    OKTA_URL: !Ref OktaUrl
                    SSM_OBJECT: !Ref SSMObjectEnv
            
    LambdaExecutionRole:
        Type: "AWS::IAM::Role"
        Properties: 
            AssumeRolePolicyDocument:
                Version: '2012-10-17'
                Statement:
                - Effect: Allow
                  Principal: 
                    Service: lambda.amazonaws.com
                  Action:
                  - sts:AssumeRole
            Path: '/'
            Policies:
            - PolicyName: GetSSMValue
              PolicyDocument:
                Version: "2012-10-17"
                Statement:
                - Effect: Allow
                  Action:
                  -  "ssm:DescribeParameters"
                  -  "ssm:GetParameter*"
                  Resource: !Sub "arn:aws:ssm:${AWS::Region}:${AWS::AccountId}:parameter${SSMObjectEnv}"
            - PolicyName: DecryptSSMValue
              PolicyDocument:
                Version: '2012-10-17'
                Statement:
                - Effect: Allow
                  Action:
                  - "kms:Decrypt"
                  Resource: !GetAtt KMSKey.Arn
            - PolicyName: "allow-lambda-logging"
              PolicyDocument:
                Version: '2012-10-17'
                Statement:
                    -   Effect: Allow
                        Action:
                            - "logs:CreateLogGroup"
                            - "logs:CreateLogStream"
                            - "logs:PutLogEvents"
                        Resource: !Sub "arn:aws:logs:${AWS::Region}:${AWS::AccountId}:*"

    KMSKey:
        Type: "AWS::KMS::Key"
        Properties:
            Description: "Key to decrypt Okta API token parameter"
            Enabled: true
            KeyPolicy:
                Version: "2012-10-17"
                Id: "okta-api-prod"
                Statement:
                    -
                        Sid: "Allow administration of the key"
                        Effect: "Allow"
                        Principal: 
                            AWS: !Sub "arn:aws:iam::${AWS::AccountId}:root"
                        Action: 
                            - "kms:Create*"
                            - "kms:Describe*"
                            - "kms:Enable*"
                            - "kms:List*"
                            - "kms:Put*"
                            - "kms:Update*"
                            - "kms:Revoke*"
                            - "kms:Disable*"
                            - "kms:Get*"
                            - "kms:Delete*"
                            - "kms:ScheduleKeyDeletion"
                            - "kms:CancelKeyDeletion"
                        Resource: "*"

    KMSAlias:
        Type: "AWS::KMS::Alias"
        Properties:
            AliasName: "alias/okta-api-kms"
            TargetKeyId: !Ref KMSKey


    OktaActivateApi:
        Type: "AWS::ApiGateway::RestApi"
        Properties:
            Name: "okta-activate-api"
            Description: "API Endpoint to trigger an Okta user activation action via Lambda"
    
    OktaActivateApiStage:
        Type: "AWS::ApiGateway::Stage"
        Properties:
            DeploymentId: !Ref OktaActivateApiDeployment
            MethodSettings:
                -   DataTraceEnabled: true
                    HttpMethod: "POST"
                    LoggingLevel: "INFO"
                    ResourcePath: "/*"
            RestApiId: !Ref OktaActivateApi
            StageName: !Ref OktaActivateApiDeployment

    OktaActivateApiDeployment:
        Type: "AWS::ApiGateway::Deployment"
        DependsOn: ["ApiPost"]
        Properties:
            RestApiId: !Ref OktaActivateApi
            StageName: "prod"

    ApiResource:
        Type: "AWS::ApiGateway::Resource"
        Properties:
            RestApiId: !Ref OktaActivateApi
            ParentId: !GetAtt ["OktaActivateApi", "RootResourceId"]
            PathPart: "okta-activate"

    ApiPost:
        Type: "AWS::ApiGateway::Method"
        Properties:
            AuthorizationType: None
            ApiKeyRequired: true
            HttpMethod: "POST"
            RestApiId: !Ref OktaActivateApi
            ResourceId: !GetAtt ["OktaActivateApi", "RootResourceId"]
            Integration: 
                Type: "AWS"
                IntegrationHttpMethod: "POST"
                Uri: !Sub arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${LambdaFunction.Arn}/invocations
                IntegrationResponses:
                    -   StatusCode: 200
                        ResponseTemplates:
                            application/json: ""
                    -   StatusCode: 403
                        SelectionPattern: ".*403.*"
                        ResponseTemplates:
                            application/json: "#set($inputRoot = $input.path('$'))\n{\n  \"message\"\
                \ : \"403 - User cannot be activated\"\n}"
                    -   StatusCode: 404
                        SelectionPattern: ".*404.*"
                        ResponseTemplates:
                            application/json: "#set($inputRoot = $input.path('$'))\n{\n  \"message\"\
                \ : \"404 - User not found!\"\n}"
                PassthroughBehavior: WHEN_NO_TEMPLATES
            MethodResponses:
                -   StatusCode: 200
                -   StatusCode: 403
                -   StatusCode: 404

    ApiUsagePlan:
        Type: "AWS::ApiGateway::UsagePlan"
        Properties:
            ApiStages:
            -   ApiId: !Ref OktaActivateApi
                Stage: !Ref OktaActivateApiStage
            UsagePlanName: "okta-activate-usage-plan"


    InvokePermission:
        Type: "AWS::Lambda::Permission"
        DependsOn: "LambdaFunction"
        Properties:
            Action: "lambda:InvokeFunction"
            FunctionName: !Ref LambdaFunction
            Principal: "apigateway.amazonaws.com"
            SourceArn: !Sub arn:aws:execute-api:${AWS::Region}:${AWS::AccountId}:${OktaActivateApi}/*/POST/

Outputs:
    KmsKeyId:
        Value: !Ref KMSKey
        Description: "ID of the KMS key"
        Export:
            Name: "okta-activate:kms:keyid"
```

The only remaining step to be completed is to create an API Key either from the AWS Console or from the `aws cli`, and associate it with our usage plan. Once that's all good you can run tests in the web console to ensure your API is working as expected. You can also test from your own machine by using `curl`. Just remember that API Gateway expects our API key to be passed in the header `x-api-key: <key>`

End User CLI App
=========
There is a huge amount of ways you could go about providing the API to your end-users, and it will all largely depend on how user friendly you want it to be and how technically apt your end user is. It could be as simple as just having them use `curl` to call the API and provide the relevant data. In my case I want this to be as convenient as possible for my end-user and so I've chosen to write a small command line tool in Go. The app expects environment variables to be locally set for both the method URL and the API Key. 

One thing to note in this code is that I am using regex to validate the user input as being an @mycompany.com email address. You will want to edit that pattern to validate user input as you need. 

```go
package main

import (
	"bufio"
	"bytes"
	"encoding/json"
	"fmt"
	"io/ioutil"
	"log"
	"net/http"
	"os"
	"regexp"
	"strings"
)

var baseURL = os.Getenv("OKTA_ACTIVATE_ENDPOINT")
var apiKey = os.Getenv("OKTA_ACTIVATE_API")

func makeRequest(url string, key string, username string) (int, string, error) {
	payloadData := map[string]string{"username": username}
	jsonPayload, _ := json.Marshal(payloadData)

	req, _ := http.NewRequest("POST", url, bytes.NewBuffer(jsonPayload))

	req.Header.Add("Content-Type", "application/json")
	req.Header.Add("x-api-key", key)

	resp, err := http.DefaultClient.Do(req)

	if err != nil {
		log.Fatal(err)
	}

	code := resp.StatusCode
	body, _ := ioutil.ReadAll(resp.Body)
	bodyString := string(body)

	return code, bodyString, err
}

func validateAddress(emailAddress string) bool {
	re := regexp.MustCompile("^[a-zA-Z0-9.!#$%&'*+/=?^_`{|}~-]+@mycompany.com*$")
	match := re.MatchString(emailAddress)
	return match
}

func main() {
	validEmail := false
	var user string
	for validEmail == false {
		reader := bufio.NewReader(os.Stdin)
		fmt.Print("Enter user to activate...\n")
		text, _ := reader.ReadString('\n')
		user = strings.TrimRight(text, "\n")
		validEmail = validateAddress(user)
	}

	code, body, err := makeRequest(baseURL, apiKey, user)
	if err != nil {
		log.Fatal(err)
	}

	if code == 200 {
		fmt.Println("User successfully activated")
	} else if code == 403 {
		fmt.Println("403 -  user may already be active or your API key may be expired")
		fmt.Println(body)
	} else if code == 404 {
		fmt.Println("404 - User not found - typo?")
		fmt.Println(body)
	}
}
```

This isn't a hugely complicated application - it simply prompts the user to enter a username to activate, validates the email address is valid for your organisation, builds a HTTP POST request, and returns the result of the request.

In our case I'm deploying the resulting Go binary to my end-user via [munki][2] - meaning I can choose exactly what machine from our fleet I want to install the binary on remotely - the user just needs to set the environment variables for the API endpoint URL and our API token.

Conclusion
====
There's a case to made that this is an overengineered solution that could've been avoided by pre-activating Okta users and doing a password reset on their account when they are ready to be onboarded, but we've worked hard at Unbounce to make the onboarding flow for new hires as convenient and positive as we can. To us it's much more preferable to have our new hires receive the onboarding email from Okta directly so there's an immediate familiarity with the service and its branding. Solving our issue this way was a fun problem to work through and is a good way to learn more about all the elements of an API driven application. 

In the future we've also allowed space for expanded capability of this tool - or porting it to a different service than Okta where we similarly run into non-granular admin permissions. 

[1]: https://docs.aws.amazon.com/lambda/latest/dg/lambda-python-how-to-create-deployment-package.html
[2]: https://github.com/munki/munki/wiki