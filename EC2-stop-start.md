# Setting up Automated Start/Stop for EC2 Instances using Terraform and AWS Lambda

## Prerequisites
1. Install Terraform on your local machine if not installed already. [Terraform Installation](https://developer.hashicorp.com/terraform/install)

2. Add the Terraform executable to your system PATH:
   - Navigate to Control Panel -> System -> System settings -> Environment Variables.
   - Scroll down to system variables, find PATH, click edit, and add the path to the Terraform executable.
   - Make sure to include a semicolon at the end of the previous entry as the delimiter (e.g., `c:\path;c:\path2`).
   - Launch a new console for the settings to take effect.

3. Open the command prompt on your local machine.

4. Execute the following command to ensure Terraform is installed on your machine:
```
$ terraform -help 
```

5. Create a new folder:
```
$ mkdir terraform-scripts
```


6. Create two Python scripts for starting and stopping the EC2 instance:

# Python script for stopping EC2 instances
a. `ec2_start.py`
```
import boto3

region = ''
instances = ['']
ec2 = boto3.client('ec2', region_name=region)

def lambda_handler(event, context):
    ec2.start_instances(InstanceIds=instances)
    print('Started your instances: ' + str(instances))
```

# Python script for stopping EC2 instances
b. `ec2_stop.py`
```
import boto3

region = ''
instances = ['']
ec2 = boto3.client('ec2', region_name=region)

def lambda_handler(event, context):
    ec2.stop_instances(InstanceIds=instances)
    print('Stopped your instances: ' + str(instances))
```
Note: Don't forget to update the instance ID and region in the Python code.

7. Now use the Terraform script below to set up Lambda functions for starting and stopping EC2 instances.

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

8. Save the file and apply the changes
```
$ terraform apply
```

9. Once done, you will see a page indicating that the two Lambda functions have been created.

   ![image](https://github.com/Jaiganesh-MJ/services-with-terraform/assets/63336185/914ff6e6-bd89-4a62-8a9c-b89636416101)

11. Now, configure EventBridge to automate the process. The following script will create a rule to start the EC2 instance at 10 AM and stop it at 6 PM.

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

11. Once created, you can see the rule updated in the Lambda function.

![image](https://github.com/Jaiganesh-MJ/services-with-terraform/assets/63336185/98d30aad-082b-45d8-9b24-91600415beae)


# Well, get ready to watch those costs shrink! 💰😄

