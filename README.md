# serverless-lab
![](.idea/images/img.png)
An Amazon API Gateway is a collection of resources and methods. I created one resource (DynamoDBManager) and define one method (POST) on it. The method is backed by a Lambda function (LambdaFunctionOverHttps). That is, when you call the API through an HTTPS endpoint, Amazon API Gateway invokes the Lambda function.

The POST method on the DynamoDBManager resource supports the following DynamoDB operations:

Create, update, delete, read and scan an item.
Other operations (echo, ping), not related to DynamoDB, that you can use for testing are also supported.
The request payload you send in the POST request identifies the DynamoDB operation and provides necessary data. For example:

The following is a sample request payload for a DynamoDB create item operation:
```json
{
    "operation": "create",
    "tableName": "lambda-apigateway",
    "payload": {
        "Item": {
            "id": "1",
            "name": "Bob"
        }
    }
}
```
The following is a sample request payload for a DynamoDB read item operation:
````json
{
    "operation": "read",
    "tableName": "lambda-apigateway",
    "payload": {
        "Key": {
            "id": "1"
        }
    }
}
````
Create a Custom Policy to add to the execution role of the Lambda function. This custom policy has the permissions that the function needs to write data to DynamoDB and upload logs:
![](.idea/images/img_1.png)
Custom Policy:
````json
{
"Version": "2012-10-17",
"Statement": [
{
  "Sid": "Stmt1428341300017",
  "Action": [
    "dynamodb:DeleteItem",
    "dynamodb:GetItem",
    "dynamodb:PutItem",
    "dynamodb:Query",
    "dynamodb:Scan",
    "dynamodb:UpdateItem"
  ],
  "Effect": "Allow",
  "Resource": "*"
},
{
  "Sid": "",
  "Resource": "*",
  "Action": [
    "logs:CreateLogGroup",
    "logs:CreateLogStream",
    "logs:PutLogEvents"
  ],
  "Effect": "Allow"
}
]
}
````


Create an Execution Role for Lambda and attach the DynamoDB_CloudWatchLogs policy.
This will give the Lambda function permission to access AWS resources.
The role should have the following properties:
   * Trusted entity – Lambda.
   * Role name – lambda-apigateway-role.
   * Permissions – Custom policy(created above) with permission to DynamoDB and CloudWatch Logs. This custom policy has the permissions that the function needs to write data to DynamoDB and upload logs.
![](.idea/images/img_2.png)

Create Lambda Function with the following code. Since we are using the Lambda function to interact with other AWS services and resources we can use Boto3(AWS SDK for Python) to interface with these resources:
![img_1.png](.idea/images/img_10.png)

![img_1.png](.idea/images/img_5.png)

```
from __future__ import print_function

import boto3
import json

print('Loading function')


def lambda_handler(event, context):
'''Provide an event that contains the following keys:

      - operation: one of the operations in the operations dict below
      - tableName: required for operations that interact with DynamoDB
      - payload: a parameter to pass to the operation being performed
    '''
    #print("Received event: " + json.dumps(event, indent=2))

    operation = event['operation']

    if 'tableName' in event:
        dynamo = boto3.resource('dynamodb').Table(event['tableName'])

    operations = {
        'create': lambda x: dynamo.put_item(**x),
        'read': lambda x: dynamo.get_item(**x),
        'update': lambda x: dynamo.update_item(**x),
        'delete': lambda x: dynamo.delete_item(**x),
        'list': lambda x: dynamo.scan(**x),
        'echo': lambda x: x,
        'ping': lambda x: 'pong'
    }

    if operation in operations:
        return operations[operation](event.get('payload'))
    else:
        raise ValueError('Unrecognized operation "{}"'.format(operation))


```
Test Lambda Function
Let's test our newly created function. We haven't created DynamoDB and the API yet, so we'll do a sample echo operation. The function should output whatever input we pass.
![img_1.png](.idea/images/img_11.png)


Deploy the code and test it with the following event:
````json
{
    "operation": "echo",
    "payload": {
        "somekey1": "somevalue1",
        "somekey2": "somevalue2"
    }
}
   ````
Click "Test", and it will execute the test event. You should see the output in the console
![](.idea/images/img_3.png)

Create a DynamoDB table and an API to invoke the Lambda function:
![](.idea/images/img_13.png)

Create an API in API Gateway:
![](.idea/images/img_14.png)
A post method has been created for the API under Resource.
![](.idea/images/img_4.png)

After creating the resource, we need to deploy the API.
The API has been deployed to a stage named Prod:
![](.idea/images/img_6.png)
Create an item in DynamoDB table using the following JSON:
````json
{
    "operation": "create",
    "tableName": "lambda-apigateway",
    "payload": {
        "Item": {
            "id": "1234ABCD",
            "number": 5
        }
    }
}
````
![](.idea/images/img_7.png)


Item is inserted into DynamoDB table:
![](.idea/images/img_8.png)

Use list to see all the items inserted into the DynamoDB table
![](.idea/images/img_9.png)