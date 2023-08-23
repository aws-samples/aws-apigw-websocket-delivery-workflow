# API Gateway WebSocket direct Integration with Step Functions and SNS

This project creates a state machine in AWS Step Functions to implement a workflow for food delivery application which can be tracked via WebSocket API. WebSocket API can be integrated in the mobile application which can send real-time update to the client.

## Project features

* This project integrates API Gateway WebSocket API directly with AWS Step Functions and Amazon SNS.
* AWS Step Functions creates a DynamoDB table to store the order details. We will be enabling TTL option on the table to store the order for 90 days to audit with Step Functions execution history.
* Websocket API will have 'sendOrder' route to trigger an order workflow. This route has [request validation](https://docs.aws.amazon.com/apigateway/latest/developerguide/websocket-api-request-validation.html) implemented on the WebSocket which validates if the request is according to the expected Model. 
* The state machine requires human approval states for various stages in the a food delivery workflow. This state machine uses Activity to achieve this. 
* This project does not implement Authentication in WebSocket API to keep it simple. However, it is recommended to implement Authentication on WebSocket API.

![Diagram](/ws-sf-sns.png)

## Pre-requisites

* [Create AWS Account](https://portal.aws.amazon.com/gp/aws/developer/registration/index.html) in case you do not have it yet or log in to an existing one
* An IAM user or a Role with sufficient permissions to deploy and manage AWS resources
* [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) installed and configured
* [Git Installed](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
* [AWS Serverless Application Model](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html) (AWS SAM) installed
* A working SNS topic which can deliver notifications to your endpoint
* An Activity in AWS Step Functions to MOCK different stages of human-approval process of food delivery

## Deployment instructions

This project contains a SAM template which can be deployed this project. You can use [wscat](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-how-to-call-websocket-api-wscat.html) to test the WebSocket. You can use AWS CLI to make GetActivityTask & SendTaskSuccess API call to progress the state machine workflow.

1. Create a new directory, navigate to that directory in a terminal and clone the GitHub repository:
```bash
git clone https://github.com/aws-samples/aws-apigw-websocket-delivery-workflow.git
```
2. Change directory to the project directory
```bash
cd aws-apigw-websocket-delivery-workflow
```
3. From the command line, use AWS SAM to deploy the AWS resources for the pattern as specified in the template.yaml file:
```bash
sam build
sam deploy -g
```
4. During the prompts for 'sam deploy -g', enter the below inputs for the prompts:
* Stack Name [sam-app]: food-delivery-app
* AWS Region [us-east-1]: us-east-1
* Parameter ApiStageName [production]: production
* Parameter Email []: <your_active_email_address>
* Confirm changes before deploy [y/N]: y
* Allow SAM CLI IAM role creation [Y/n]: Y
* Disable rollback [y/N]: y
* Save arguments to configuration file [Y/n]: Y
* SAM configuration file [samconfig.toml]:
* SAM configuration environment [default]:

In the midst of deployment, you should receive an email to confirm the SNS subscription. The template configures your provided email address to an SNS topic to receive the delivery notification.

## Testing

1. The stack will output the **api endpoint**. Use wscat to test the API (see [documentation](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-how-to-call-websocket-api-wscat.html) for more details on how to set it up):

```bash
wscat -c < API ENdpoint from the stack >
```
2. Send a payload to the API by typing it in. We need to send the payload in the right format, otherwise you will receive a 'Forbidden' exception:
```bash
> {"action":"sendOrder","orderDetails":"daal makhanix1,garlic naan x1,raitax2","customerId":"2211","restaurantDetails":"NewDeliRestaurant"}
< {"executionArn":"arn:aws:states:us-east-1:123456789012:execution:orderWorkflow-9GwbXOgAhILw:8b639bdd-0cbf-4a33-af8b-e04f2d3dd327","startDate":1.686509515191E9}
< {"Payload":"Order initiated! Preparing daal makhanix1,garlic naan x1,raitax2 and looking for a delivery partner!"}
< {"Payload":"Order daal makhanix1,garlic naan x1,raitax2 prepared! It will reach you soon."}
< {"Payload":"Delivery partner Jon assigned!"}
< {"Payload":"Tasty food is on its way! Jon is bringing daal makhanix1,garlic naan x1,raitax2!"}
< {"Payload":"daal makhanix1,garlic naan x1,raitax2 delivered by Jon!"}
```
3. During the workflow, you would need to use [GetActivityTask](https://docs.aws.amazon.com/step-functions/latest/apireference/API_GetActivityTask.html) and [SendTaskSuccess](https://docs.aws.amazon.com/step-functions/latest/apireference/API_SendTaskSuccess.html) API calls for the activity you provided to the template in order to progress the workflow. Make sure to pass the Delivery Agent name in the --task-output parameter when in 'findDeliveryPartner' state of your workflow execution. Follow the below API calls to progress the execution:

TO MARK THE FOOD PREPARATION AS COMPLETE
```bash
$ aws stepfunctions get-activity-task --activity-arn arn:aws:states:us-east-1:123456789012:activity:prepareFoodActivity
$ aws stepfunctions send-task-success  --task-output {}  --task-token <TASK_TOKEN_RECEIVED_FROM_PREVIOUS_STEP>
```
TO FIND DELIVERY PARTNER
```bash
$ aws stepfunctions get-activity-task --activity-arn arn:aws:states:us-east-1:123456789012:activity:findDeliveryPartnerActivity
$ aws stepfunctions send-task-success  --task-output {\"name\":\"Jon\"} --task-token <TASK_TOKEN_RECEIVED_FROM_PREVIOUS_STEP>
```
TO TRACK DELIVERY STATUS
```bash
$ aws stepfunctions get-activity-task --activity-arn arn:aws:states:us-east-1:123456789012:activity:deliveryStatusActivity
$ aws stepfunctions send-task-success  --task-output {}  --task-token <TASK_TOKEN_RECEIVED_FROM_PREVIOUS_STEP>
```

## Cleanup
 
1. Delete the stack
    ```bash
    sam delete
    ```
2. Confirm the stack has been deleted
    ```bash
    aws cloudformation list-stacks --query "StackSummaries[?contains(StackName,'STACK_NAME')].StackStatus"
    ```
----
Copyright 2023 Amazon.com, Inc. or its affiliates. All Rights Reserved.

SPDX-License-Identifier: MIT-0