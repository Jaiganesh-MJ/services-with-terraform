I'd like to share a document, especially relevant for DevOps engineers, as many of us have encountered questions from clients about high AWS charges for resources. After analyzing the situation, we devised a plan to remove unused resources from AWS. In some cases, we identified resources that weren't needed 24/7, such as non-production environments or applications hosted in Docker containers running on EC2 instances.

For instance, these environments may only be necessary during working hours, say from 9 AM to 5 PM. During other times, we can optimize costs by stopping the virtual machines. I implemented this setup during my initial stages, and upon reflection, I realized it took a considerable amount of time to set up for another project. To streamline the process, I decided to use Terraform. In this example, I demonstrate how to stop an EC2 instance, but a similar approach can be applied to other resources. Below are the steps to configure this using Terraform.

Feel free to share your thoughts or ask any questions!


1. Install terraform in your local machine if not installed already
   https://developer.hashicorp.com/terraform/install

2. Add the Terraform executable to your system PATH: 
- Go to Control Panel -> System -> System settings -> Environment Variables. 
- Scroll down in system variables until you find PATH. 
- Click edit and add the path to the Terraform executable. 
- Make sure to include a semicolon at the end of the previous entry as the delimiter (e.g., c:\path;c:\path2). 
- Launch a new console for the settings to take effect. 

3. Open the command prompt on your local machine.
   
4. Execute the following command to ensure Terraform is installed on your machine: 
$ terraform -help 

6. Create a new folder: 
$ mkdir terraform-scripts

7. Create two python scripts for stop and start the EC2 instance
a. ec2_start.py
```
import boto3
import os
region = 'eu-west-1'
instances = ['i-0c11cc639dd627491']
ec2 = boto3.client('ec2', region_name=region)

def lambda_handler(event, context):
    ec2.start_instances(InstanceIds=instances)
    print('started your instances: ' + str(instances))
```

b. ec2_stop.py
```
import boto3
import os
region = ''
instances = ['']
ec2 = boto3.client('ec2', region_name=region)

def lambda_handler(event, context):
    ec2.stop_instances(InstanceIds=instances)
    print('stopped your instances: ' + str(instances))
```

Note: Don't forgot to update instance id and region in above python code

8. Now use the below terraform script to setup lambda function, this includes creation of IAM policy, lambda fucntion with python3.9 along with the code that we mentioned before.

```
variable "region" {}

variable "instance_id" {}

resource "aws_iam_policy" "policy" {
  name        = "ec2_stop_start_policy"
  path        = "/"
  description = "ec2_stop_start_policy"

  policy = jsonencode({
    "Version" : "2012-10-17",
    "Statement" : [
      {
        "Effect" : "Allow",
        "Action" : [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ],
        "Resource" : "arn:aws:logs:*:*:*"
      },
      {
        "Effect" : "Allow",
        "Action" : [
          "ec2:Start*",
          "ec2:Stop*"
        ],
        "Resource" : "*"
      }
    ]
  })
}

resource "aws_iam_policy_attachment" "ec2_auto_attach" {
  name       = "ec2_stop_start_policy_attachment"
  roles      = [aws_iam_role.lambda_role.name]
  policy_arn = aws_iam_policy.policy.arn
}

resource "aws_iam_role" "lambda_role" {
  name = "lambda_execution_role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect = "Allow",
        Principal = {
          Service = "lambda.amazonaws.com"
        },
        Action = "sts:AssumeRole"
      }
    ]
  })
}

# Attach the policy to the IAM role
resource "aws_iam_role_policy_attachment" "lambda_policy_attachment" {
  policy_arn = aws_iam_policy.policy.arn
  role       = aws_iam_role.lambda_role.name
}

data "archive_file" "souece_ec2_stop" {
  type        = "zip"
  source_file = "/workspaces/my-terraform-journey/modules_practice/ec2_create/ec2_stop.py"
  output_path = "ec2_stop.zip"
}

resource "aws_lambda_function" "ec2_stop" {

  filename      = "ec2_stop.zip"
  function_name = "ec2_stop_auto"
  role          = aws_iam_role.lambda_role.arn
  handler       = "ec2_stop.lambda_handler"
  timeout       = 60

  source_code_hash = data.archive_file.souece_ec2_stop.output_base64sha256

  runtime = "python3.9"

  environment {
    variables = {
      REGION      = var.region
      INSTANCE_ID = var.instance_id
    }
  }
}


data "archive_file" "souece_ec2_start" {
  type        = "zip"
  source_file = "/workspaces/my-terraform-journey/modules_practice/ec2_create/ec2_start.py"
  output_path = "ec2_start.zip"
}

resource "aws_lambda_function" "ec2_start" {

  filename      = "ec2_start.zip"
  function_name = "ec2_start_auto"
  role          = aws_iam_role.lambda_role.arn
  handler       = "ec2_start.lambda_handler"
  timeout       = 60

  source_code_hash = data.archive_file.souece_ec2_start.output_base64sha256

  runtime = "python3.9"


  environment {
    variables = {
      REGION      = var.region
      INSTANCE_ID = var.instance_id
    }
  }
}
```

9. Save the file and apply the changes
```
$ terraform apply
```

10. Once it done you will be see the similar page like below, means two lambda functions are created.
    ![image](https://github.com/Jaiganesh-MJ/services-with-terraform/assets/63336185/914ff6e6-bd89-4a62-8a9c-b89636416101)

11. Now time to configure event bridge to automate the process. The below script will help to launch an bridge rule and it will start the EC2 instance at 10AM and stop at 6PM

```
resource "aws_cloudwatch_event_rule" "morning_rule" {
  name        = "morning_rule"
  description = "Rule to trigger Lambda function at 10 AM"
  schedule_expression = "cron(0 10 * * ? *)"

}

resource "aws_cloudwatch_event_rule" "evening_rule" {
  name        = "evening_rule"
  description = "Rule to trigger Lambda function at 6 PM"
  schedule_expression = "cron(0 18 * * ? *)"

}

resource "aws_cloudwatch_event_target" "morning_lambda_target" {
  rule      = aws_cloudwatch_event_rule.morning_rule.name
  arn       = aws_lambda_function.ec2_start.arn
  target_id = "morning_lambda_target"
}

resource "aws_cloudwatch_event_target" "evening_lambda_target" {
  rule      = aws_cloudwatch_event_rule.evening_rule.name
  arn       = aws_lambda_function.ec2_stop.arn
  target_id = "evening_lambda_target"
}


resource "aws_lambda_permission" "ec2_start_perm" {
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.ec2_start.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.morning_rule.arn
}

resource "aws_lambda_permission" "ec2_stop_perm" {
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.ec2_stop.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.evening_rule.arn
}
```

12. Once created we can see the rule updated in lambda
![image](https://github.com/Jaiganesh-MJ/services-with-terraform/assets/63336185/98d30aad-082b-45d8-9b24-91600415beae)
