---
title: Serverless and Node.js
date: 2019-07-09 17:04:39
tags: ['Serverless', 'Node.js']
---

随着技术的发展，有一个目标总是不变的——让一切变得更加简单。在开发传统的B/S架构应用时，需要自行部署服务器、数据库，配置CI/CD等一系列繁琐的事情，往往需要一整个团队的配合。而在Serverless架构中，其试图帮助开发者摆脱运行后端应用程序所需的服务器设备的设置和管理工作，将全部的注意力集中在核心业务逻辑中。通过配合API Gateway、S3、Aurora等即可完成应用的构建。

在aws官方推荐中，一个FAAS的serverless function的生存时间一般是3s到5min。因此需要更快的初始化加载速度和敏捷的开发速度，从而促使Node.js在这一领域大放异彩。这是一个系列博客，主要讲如何将serverless架构的API部署到AWS，并通过S3、DynamoDB来提供持久化。

# Step 1 Create IAM User

在AWS console中，在`安全性、身份与合规性`下进入`IAM`，在创建一个新的用户，访问类型设为`编程访问`，在权限设置中选择`直接附加现有策略`中的`AdministratorAccess`，然后完成IAM的创建。IAM将会用于AWS service之间的调用时的身份验证。

# Step 2 Create DynamoDB Table

创建一个DynamoDB的表，名为notes，主键为userId，搜索键为noteId，配置时不使用默认配置，改为按需，然后一个表就创建好了。

# Step 3 Create S3 bucket

全部选项设为默认即可。然后对该bucket的权限配置CORS，添加以下配置：
```
<CORSConfiguration>
	<CORSRule>
		<AllowedOrigin>*</AllowedOrigin>
		<AllowedMethod>GET</AllowedMethod>
		<AllowedMethod>PUT</AllowedMethod>
		<AllowedMethod>POST</AllowedMethod>
		<AllowedMethod>HEAD</AllowedMethod>
		<AllowedMethod>DELETE</AllowedMethod>
		<MaxAgeSeconds>3000</MaxAgeSeconds>
		<AllowedHeader>*</AllowedHeader>
	</CORSRule>
</CORSConfiguration>
```

# Step 4 Create Cognito User Pool

为了处理用户账户登录验证等，需要创建一个Cognito User Pool。这里进入Cognito后选择`管理用户池`，然后`创建用户池`，通过`查看默认值`方式创建，将`用户名属`性选为`电子邮件`,然后即可创建用户池。然后选择添加一个应用程序客户端，不勾选`生成客户端密钥`，选择`为基于服务器的身份验证启用登录 API (ADMIN_NO_SRP_AUTH)`。然后在`域名`中输入一个域名即可。

这里首先创建一个测试账户：
```
aws cognito-idp sign-up \
  --region YOUR_COGNITO_REGION \
  --client-id YOUR_COGNITO_APP_CLIENT_ID \
  --username admin@example.com \
  --password Passw0rd!
```
并得到类似以下回复：
```
{
    "UserConfirmed": false, 
    "UserSub": "4bfff1d9-afa3-4e88-944a-5a7719793c80", 
    "CodeDeliveryDetails": {
        "AttributeName": "email", 
        "Destination": "lzkmylz@163.com", 
        "DeliveryMedium": "EMAIL"
    }
}
```

然后管理一个身份池，将身份验证提供商设为Cognito，填写用户池ID和应用程序客户端ID，完成创建。然后展开详细信息并编辑策略文档：
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "mobileanalytics:PutEvents",
        "cognito-sync:*",
        "cognito-identity:*"
      ],
      "Resource": [
        "*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:*"
      ],
      "Resource": [
        "arn:aws:s3:::YOUR_S3_UPLOADS_BUCKET_NAME/private/${cognito-identity.amazonaws.com:sub}/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "execute-api:Invoke"
      ],
      "Resource": [
        "arn:aws:execute-api:YOUR_API_GATEWAY_REGION:*:YOUR_API_GATEWAY_ID/*/*/*"
      ]
    }
  ]
}
```
进入控制面板并编辑身份池，

# Step 5 Setup Serverless Framework

首先进行安装serverless package
```
npm install serverless -g
```

在一个文件夹内初始化项目：
```
serverless install --url https://github.com/AnomalyInnovations/serverless-nodejs-starter --name my-project // 添加ES6/ES7支持
npm install
npm install aws-sdk --save-dev
npm install uuid --save
```

# Step 6 Write Backend API

* Create Note API

创建文件create.js，在这里加入创建Note的API。

```
import uuid from "uuid";
import AWS from "aws-sdk";

const dynamoDb = new AWS.DynamoDB.DocumentClient();

export function main(event, context, callback) {
  // Request body is passed in as a JSON encoded string in 'event.body'
  const data = JSON.parse(event.body);

  const params = {
    TableName: "notes",
    // 'Item' contains the attributes of the item to be created
    // - 'userId': user identities are federated through the
    //             Cognito Identity Pool, we will use the identity id
    //             as the user id of the authenticated user
    // - 'noteId': a unique uuid
    // - 'content': parsed from request body
    // - 'attachment': parsed from request body
    // - 'createdAt': current Unix timestamp
    Item: {
      userId: event.requestContext.identity.cognitoIdentityId,
      noteId: uuid.v1(),
      content: data.content,
      attachment: data.attachment,
      createdAt: Date.now()
    }
  };

  dynamoDb.put(params, (error, data) => {
    // Set response headers to enable CORS (Cross-Origin Resource Sharing)
    const headers = {
      "Access-Control-Allow-Origin": "*",
      "Access-Control-Allow-Credentials": true
    };

    // Return status code 500 on error
    if (error) {
      const response = {
        statusCode: 500,
        headers: headers,
        body: JSON.stringify({ status: false })
      };
      callback(null, response);
      return;
    }

    // Return status code 200 and the newly created item
    const response = {
      statusCode: 200,
      headers: headers,
      body: JSON.stringify(params.Item)
    };
    callback(null, response);
  });
}
```

然后在serverless.yml中加入配置：
```
provider:
  name: aws
  runtime: nodejs8.10
  stage: prod
  region: us-east-1

  # 'iamRoleStatements' defines the permission policy for the Lambda function.
  # In this case Lambda functions are granted with permissions to access DynamoDB.
  iamRoleStatements:
    - Effect: Allow
      Action:
        - dynamodb:DescribeTable
        - dynamodb:Query
        - dynamodb:Scan
        - dynamodb:GetItem
        - dynamodb:PutItem
        - dynamodb:UpdateItem
        - dynamodb:DeleteItem
      Resource: "arn:aws:dynamodb:us-east-1:*:*"

functions:
  # Defines an HTTP API endpoint that calls the main function in create.js
  # - path: url path is /notes
  # - method: POST request
  # - cors: enabled CORS (Cross-Origin Resource Sharing) for browser cross
  #     domain api call
  # - authorizer: authenticate using the AWS IAM role
  create:
    handler: create.main
    events:
      - http:
          path: notes
          method: post
          cors: true
          authorizer: aws_iam
```

创建用于测试的mock json文件

```
{
  "body": "{\"content\":\"hello world\",\"attachment\":\"hello.jpg\"}",
  "requestContext": {
    "identity": {
      "cognitoIdentityId": "USER-SUB-1234"
    }
  }
}
```

然后测试API调用
```
serverless invoke local --function create --path mocks/create-event.json
```

这将会回复类似以下的内容：
```
{
  statusCode: 200,
  headers: {
    'Access-Control-Allow-Origin': '*',
    'Access-Control-Allow-Credentials': true
  },
  body: '{"userId":"USER-SUB-1234","noteId":"578eb840-f70f-11e6-9d1a-1359b3b22944","content":"hello world","attachment":"hello.jpg","createdAt":1487800950620}'
}
```

随后可以部署API：
```
serverless deploy
```

可以看到类似以下的回应：
```
service: lambda-test
stage: prod
region: ap-southeast-1
stack: lambda-test-prod
resources: 11
api keys:
  None
endpoints:
  POST - https://i2qbe2fm8d.execute-api.ap-southeast-1.amazonaws.com/prod/notes
functions:
  create: lambda-test-prod-create
layers:
  None

```

此时可以测试刚刚部署的API：
```
npx aws-api-gateway-cli-test
npx aws-api-gateway-cli-test \
--username='admin@example.com' \
--password='Passw0rd!' \
--user-pool-id='YOUR_COGNITO_USER_POOL_ID' \
--app-client-id='YOUR_COGNITO_APP_CLIENT_ID' \
--cognito-region='YOUR_COGNITO_REGION' \
--identity-pool-id='YOUR_IDENTITY_POOL_ID' \
--invoke-url='YOUR_API_GATEWAY_URL' \
--api-gateway-region='YOUR_API_GATEWAY_REGION' \
--path-template='/notes' \
--method='POST' \
--body='{"content":"hello world","attachment":"hello.jpg"}'
```

至此一个示例API就已经完成了。