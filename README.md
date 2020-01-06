
## 3.1 创建Lambda任务

### 3.1.1 Lottery-InputWinners-\<username>  函数

```
import json

class CustomError(Exception):
    pass

def lambda_handler(event, context):
    num_of_winners = event['input']
    
    # Trigger the Failed process
    if 'exception' in event:
        raise CustomError("An error occurred!!")
    
    return {
        "body": {
            "num_of_winners": num_of_winners
        }
    }
```

### 3.1.2 Lottery-RandomSelectWinners-\<username> 函数

```
import json
import boto3
from random import randint
from boto3.dynamodb.conditions import Key, Attr

TOTAL_NUM = 10

def lambda_handler(event, context):
    # variables
    num_of_winners = event['num_of_winners']
    
    # query in dynamodb
    dynamodb = boto3.resource('dynamodb', region_name='cn-northwest-1')
    table = dynamodb.Table('Lottery-Employee-<username>')

    # random select the winners, if has duplicate value, re-run the process
    while True:
        lottery_serials = [randint(1,TOTAL_NUM) for i in range(num_of_winners)]
        if len(lottery_serials) == len(set(lottery_serials)):
            break
    
    # retrieve the employee details from dynamodb
    results = [table.query(KeyConditionExpression=Key('lottery_serial').eq(serial), IndexName='lottery_serial-index') for serial in lottery_serials]
    
    # format results
    winner_details = [result['Items'][0] for result in results]
    
    return {
        "body": {
            "num_of_winners": num_of_winners,
            "winner_details": winner_details
        }
    }
```

### 3.1.3 Lottery-ValidateWinners-\<username> 函数

```
import json
import boto3
from boto3.dynamodb.conditions import Key, Attr

def lambda_handler(event, context):
    # variables
    num_of_winners = event['num_of_winners']
    winner_details = event['winner_details']
    
    # query in dynamodb
    dynamodb = boto3.resource('dynamodb', region_name='cn-northwest-1')
    table = dynamodb.Table('Lottery-Winners-*<username>*')

    # valiate whether the winner has already been selected in the past draw
    winners_employee_id = [winner['employee_id'] for winner in winner_details]
    results = [table.query(KeyConditionExpression=Key('employee_id').eq(employee_id)) for employee_id in winners_employee_id]
    output = [result['Items'] for result in results if result['Count'] > 0]
    
    # if winner is in the past draw, return 0 else return 1
    has_winner_in_queue = 1 if len(output) > 0 else 0
    
    # format the winner details in sns
    winner_names = [winner['employee_name'] for winner in winner_details]
    
    name_s = ""
    for name in winner_names:
        name_s += name
        name_s += " "
        
    return {
        "body": {
            "num_of_winners": num_of_winners,
            "winner_details": winner_details
        },
        "status": has_winner_in_queue,
        "sns": "Congrats! [{}] You have selected as the Lucky Champ!".format(name_s.strip())
    }
```

### 3.1.4 Lottery-ValidateWinners-\<username> 函数

```
import json
import boto3
from boto3.dynamodb.conditions import Key, Attr

def lambda_handler(event, context):
    # variables
    winner_details = event['winner_details']
    
    # retrieve the winners' employee id
    employee_ids = [winner['employee_id'] for winner in winner_details]
    
    # save the records in dynamodb
    dynamodb = boto3.resource('dynamodb', region_name='cn-northwest-1')
    table = dynamodb.Table('Lottery-Winners-<username>')
    
    for employee_id in employee_ids:
        table.put_item(Item={
            'employee_id': employee_id
        })
        
    return {
        "body": {
            "winners": winner_details
        },
        "status_code": "SUCCESS" 
    }
```

## 3.4 Stepfunction 状态定义文件

```
{
  "Comment": "A simple AWS Step Functions state machine that simulates the lottery session",
  "StartAt": "Input Lottery Winners",
  "States": {
    "Input Lottery Winners": {
        "Type": "Task",
        "Resource": "<InputWinners:ARN>",
        "ResultPath": "$",
        "Catch": [ 
            {          
              "ErrorEquals": [ "CustomError" ],
              "Next": "Failed"      
            },
            {          
              "ErrorEquals": [ "States.ALL" ],
              "Next": "Failed"      
            } 
          ],
        "Next": "Random Select Winners"
    }, 
    "Random Select Winners": {
      "Type": "Task",
      "InputPath": "$.body",
      "Resource": "<RandomSelectWinners:ARN>",
      "Catch": [ 
        {          
          "ErrorEquals": [ "States.ALL" ],
          "Next": "Failed"      
        } 
      ],      
     "Retry": [ 
        {
          "ErrorEquals": [ "States.ALL"],          
          "IntervalSeconds": 1, 
          "MaxAttempts": 2
        } 
      ],
      "Next": "Validate Winners"
    },
    "Validate Winners": {
      "Type": "Task",
      "InputPath": "$.body",
      "Resource": "<ValidateWinners:ARN>",
      "Catch": [ 
        {          
          "ErrorEquals": [ "States.ALL" ],
          "Next": "Failed"      
        } 
      ],      
     "Retry": [ 
        {
          "ErrorEquals": [ "States.ALL"],          
          "IntervalSeconds": 1, 
          "MaxAttempts": 2
        } 
      ],
      "Next": "Is Winner In Past Draw"
    },
    "Is Winner In Past Draw": {
      "Type" : "Choice",
        "Choices": [
          {
            "Variable": "$.status",
            "NumericEquals": 0,
            "Next": "Send SNS and Record In Dynamodb"
          },
          {
            "Variable": "$.status",
            "NumericEquals": 1,
            "Next": "Random Select Winners"
          }
      ]
    },
    "Send SNS and Record In Dynamodb": {
      "Type": "Parallel",
      "End": true,
      "Catch": [ 
        {          
          "ErrorEquals": [ "States.ALL" ],
          "Next": "Failed"      
        } 
      ],      
     "Retry": [ 
        {
          "ErrorEquals": [ "States.ALL"],          
          "IntervalSeconds": 1, 
          "MaxAttempts": 2
        } 
      ],
        "Branches": [
        {
         "StartAt": "Notify Winners",
         "States": {
           "Notify Winners": {
             "Type": "Task",
             "Resource": "arn:aws-cn:states:::sns:publish",
             "Parameters": {
               "TopicArn": "<Notification:ARN>",
               "Message.$": "$.sns"
             },
             "End": true
           }
         }
       },
          {
         "StartAt": "Record Winner Queue",
         "States": {
           "Record Winner Queue": {
             "Type": "Task",
             "InputPath": "$.body",
             "Resource":
               "<RecordWinners:ARN>",
             "TimeoutSeconds": 300,
             "End": true
           }
         }
       }
      ]
    },
    "Failed": {
        "Type": "Fail"
     }
  }
}
```


