🚀 Sharing Knowledge: Streaming ECS Container Logs to CloudWatch 🚀

Hey LinkedIn community! 👋 Excited to share a quick tutorial on how to seamlessly stream logs from your ECS container to CloudWatch. Whether you're working with AWS, Terraform, or just diving into container orchestration, this step-by-step guide can be a handy reference. 🌐

Step 1: Set Up ECS Container 🛠️
Ensure your ECS container is configured. If not, no worries! I've included a Terraform snippet to help set up your ECS cluster. It even comes with a launch configuration, autoscaling group, and an IAM role for good measure.

data "aws_ami" "ecs_ami" {
  most_recent = true

  filter {
    name   = "name"
    values = ["amzn2-ami-ecs-hvm-2.0.20230109-x86_64-ebs"]
  }

  owners = ["amazon"]
}

# Create an ECS cluster
resource "aws_ecs_cluster" "my_cluster" {
  name = var.environment

  setting {
    name  = "containerInsights"
    value = "enabled"
  }

}

resource "aws_iam_role" "ecs_instance_role" {
  name = var.env_instance_profile
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })
}


# Create an IAM instance profile for ECS instances
resource "aws_iam_instance_profile" "ecs_instance_profile" {
  name = var.env_instance_profile
  role = aws_iam_role.ecs_instance_role.name
}

resource "aws_iam_role_policy_attachment" "ecs_instance_policy" {
  policy_arn = var.policy_arn
  role       = aws_iam_role.ecs_instance_role.name
}

# Create a launch configuration for the EC2 instances
resource "aws_launch_configuration" "my_launch_config" {
  name_prefix   = var.launch_config_name
  image_id      = data.aws_ami.ecs_ami.id
  instance_type = var.server_type
  associate_public_ip_address = true
  iam_instance_profile = aws_iam_instance_profile.ecs_instance_profile.name
  key_name      = var.key_name
  security_groups = [var.sg_id]

  user_data = <<-EOF
              #!/bin/bash
              echo ECS_CLUSTER=dev-2 >> /etc/ecs/ecs.config
              yum update -y
              yum install -y aws-cli
                    echo ECS_BACKEND_HOST= >> /etc/ecs/ecs.config
              EOF

  lifecycle {
    create_before_destroy = true
  }
}

# Create an auto scaling group to launch the EC2 instances
resource "aws_autoscaling_group" "my_asg" {
  name                 = var.environment
  max_size             = 1
  min_size             = 1
  desired_capacity     = 1
  health_check_grace_period = 300

  launch_configuration = aws_launch_configuration.my_launch_config.id

  vpc_zone_identifier = var.subnets

  tag {
    key                 = "Name"
    value               = var.environment
    propagate_at_launch = true
  }
}

# Create a task definition for the service
resource "aws_ecs_task_definition" "my_task_definition" {
  family                   = var.environment
  network_mode             = "bridge"
  cpu                      = var.cpu
  memory                   = var.memory
  container_definitions    = jsonencode([
    {
      name            = var.environment
      image           = var.application_image
      memory          = var.memory
      portMappings    = [
        {
          containerPort = 443
          hostPort      = 0
        }
      ]

    "mountPoints": [
      {
        "sourceVolume": "nginx-logs",
        "containerPath": "/var/log/nginx/"
      }
    ]

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group" = var.awslogs_group
          "awslogs-stream-prefix" = var.awslogs_stream_prefix
          "awslogs-region" = var.aws_region
        }
      }
    }
  ])


    volume {
        name = "nginx-logs"
        host_path = "/home/ec2-user/my-logs/"
    }


  requires_compatibilities = ["EC2"]
}


# Create a new Application Load Balancer
resource "aws_lb" "ecs_lb" {
  name               = var.environment
  internal           = false
  load_balancer_type = "application"

  subnets         = var.subnets
  security_groups = [var.lb_sg_id]

  tags = {
    Name = var.alb_name
  }
}

# Create a listener on the ALB
resource "aws_lb_listener" "ecs_lb_listener" {
  load_balancer_arn = aws_lb.ecs_lb.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-2016-08"

  certificate_arn   = var.acm_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.ecs_target_group.arn
  }
}

# Create a target group for the ECS service
resource "aws_lb_target_group" "ecs_target_group" {
  name        = var.ecs_target_group
  port        = 443
  protocol    = "HTTPS"
  vpc_id      = var.vpc
  target_type = "instance"
  health_check {
    path             = "/"
    interval         = 30
    timeout          = 5
    protocol         = "HTTPS"
  }
}

# Create an ECS service to run the task definition
resource "aws_ecs_service" "my_service" {
  name            = var.environment
  cluster         = aws_ecs_cluster.my_cluster.id
  task_definition = aws_ecs_task_definition.my_task_definition.arn
  launch_type     = "EC2"

  # Attach the ALB as the target group for this service
  load_balancer {
    target_group_arn = aws_lb_target_group.ecs_target_group.arn
    container_name   = var.environment
    container_port   = 443
  }


  deployment_controller {
    type = "ECS"
  }

  desired_count = 1
}

Step 2: Install CloudWatch Agent ☁️

In the next step, we will install the CloudWatch agent on the container instance. While it's possible to install it along with user data, this guide will cover the manual installation process to provide a clear understanding of the involved commands.

a. SSH into the container instance using the command below:
$ ssh -i "key.pem" user_name@dns_name

b. Once logged into the server, create a directory to store agent packages:
$ mkdir cloudwatch
$ cd cloudwatch

c. Download the CloudWatch agent package using the following command:
$ wget https://amazoncloudwatch-agent.s3.amazonaws.com/amazon_linux/amd64/latest/amazon-cloudwatch-agent.rpm

d. Install the Amazon CloudWatch Agent on the system using the RPM package manager:
$ sudo rpm -U ./amazon-cloudwatch-agent.rpm

e. Now that the CloudWatch agent is installed, it's time to update the logs to be monitored. Open the config.json file and update the configurations:
$ vi /opt/aws/amazon-cloudwatch-agent/bin/config.json

f. Add the necessary log configurations. In this example, monitoring only nginx logs is demonstrated:
{
  "agent": {
    "metrics_collection_interval": 10,
    "run_as_user": "root"
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/home/ec2-user/my-logs/access.log",
            "log_group_name": "dev2-nginx-access-log",
            "log_stream_name": "dev2-nginx-access-log",
            "timezone": "Local"
          },
          {
            "file_path": "/home/ec2-user/my-logs/error.log",
            "log_group_name": "dev2-nginx-error-log",
            "log_stream_name": "dev2-nginx-error-log",
            "timezone": "Local"
          }
        ]
      }
    }
  }
}



g. Sync the CloudWatch config with the agent:
$ sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json -s


h. Check the agent status using the following command:
$ sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status


Step 3: Mount ECS Container Instance Path 🗂️

Now, it's time for the main scene! Mount the ECS container instance path with the container. Below is the Terraform code to update your configuration. I've already completed these steps when creating the ECS configuration, but I'm providing the code for reference.

Add mountpoints inside container definitions:
"mountPoints": [
  {
    "sourceVolume": "nginx-logs",
    "containerPath": "/var/log/nginx/"
  }
]

	
Add volume inside task definition creation:
volume {
    name      = "nginx-logs"
    host_path = "/home/ec2-user/my-logs/"
}


So, the mountpoint should be the path of logs in the container, and the volume host path is where the logs need to be synced.


Additionally, I'm mentioning the manual steps to configure volume in ECS.

a. Open the necessary task definition that we need to configure volume.
b. Scroll down to the end of the page. Here, we can add a volume.
c. Click "Add volume," enter the name of the volume, volume type, and source path.
d. In the mount point, choose the container, source volume, and container path.
e. Once this is done, click create to update the settings.
We've created a new version of the task definition. Now, we need to update the service to reflect the changes. To do that, execute the following AWS CLI command.
$ aws ecs update-service --cluster dev-2 --service dev-2 --force-new-deployment --region $region


Once everything is done, you can check logs in CloudWatch within the specific log group we mentioned.
