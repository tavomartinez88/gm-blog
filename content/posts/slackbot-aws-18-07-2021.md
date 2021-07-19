---
title: "How do you make a bot using Aws?"
date: 2021-07-18
draft: true
tags: ["Aws", "Go", "Slack", "Terraform"]    
---

## Introduction
>
> Let's go build a bot to inform new releases using Go as language programming,
> Aws as infraestructure on cloud, Terraform as infraestructure as code.
>
## Requirements

* An account on Aws
* Go 1.15 or greather installed
* Terraform 0.13.2 or greather installed
* Gomock 1.6.0 or greather downloaded
* Git and Github/GitLab/Bitbucket/Other 

## The follow picture is our work finished
![Alt text](https://raw.githubusercontent.com/tavomartinez88/slack-bot/main/docs/Slack-bot.png)

## Let's Go!!!!
>
>The fisrt we are need define structure to our project, this is :
> - dynamodb
>    - mock_dynamodb(here put all mock files to dynamodb)
> - lambda
>    - hanlder
>    - models
>    - processor
>    - utils
> - terraform
> - docs (here locate images and diagrams)  
>
>Well, now let's go initialize the module go, because we going to initialize repository, for it, you should run the 
>next command :
>
<pre>
    go mod init github.com/username/slack-bot
</pre>
>
>Then we are going to create the golang code for the lambda, the most important are on the folders handler and processor
> , but if you want watch more details about it, ypu can go to [repo](https://github.com/tavomartinez88/slack-bot).
>On folder hanlder do yoy should create a file named Handler.go
>
<pre>
const (
	FormatDateTimecategory = "01-02-2006 15:04:05"
)

func HandleRequest(ctx context.Context, request models.Request) error {
	slackBotDb := dynamodb.GetSlackBotDb()
	processor := p.NewProcessor(slackBotDb)
	err := processor.Process(uuid.New().String(), time.Now().Format(FormatDateTimecategory), request)

	if err != nil {
		return err
	}

	return nil
}
</pre>
>
>Next, do you should create on folder processor a file named processor.go
>
<pre>
type Processor struct {
	SlackBotDb dynamodb.IDynamoSlackBotDb
}

func NewProcessor(i dynamodb.IDynamoSlackBotDb) *Processor {
	return &Processor{i}
}

func (receiver *Processor) Process(id string, createdDate string, request models.Request) error {
	logger.Info("Start Process")

	_, err := utils.ValidInput(request)

	if err != nil {
		logger.Error("Invalid request", err)
		return err
	}

	request.Id = id
	request.CreateDate = createdDate

	logger.Println("Send request to create release")

	err = receiver.SlackBotDb.CreateRelease(request)

	if err != nil {
		logger.Error("Error trying create release on dynamodb", err)
		logger.Info("Finished Process")
		return err
	}

	logger.Println("Send request to create release successful")
	logger.Info("Finished Process")
	return nil
}

</pre>

>
>After that, we are need create some resources on aws as table on dynamo db, for it do you should create file 
>named main.tf on folder terraform/main.tf and place it :
>
<pre>
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 3.27"
    }
  }
}

provider "aws" {
  profile = ""
  region  = ""
}
</pre>
>
>This is for define the cloud provider and provider info as your profile and region. After that add the follow code 
>to create variables required and the table.
>
<pre>
//------------------------------------------
//  VARIABLES
//------------------------------------------
variable "slack-bot-release-app-name" {
  description = "Lambda Notify Releases"
  default = "slack-release-service"
}

variable "app_env" {
  description = "Application environment tag"
  default     = "dev"
}

locals {
  app_id = "${lower(var.slack-bot-release-app-name)}-${lower(var.app_env)}"
}

//------------------------------------------
//  DYNAMODB
//------------------------------------------

resource "aws_dynamodb_table" "slack-bot" {
  name           = "Releases"
  billing_mode   = "PAY_PER_REQUEST"
  read_capacity  = 20
  write_capacity = 20
  hash_key       = "id"
  stream_enabled   = true
  stream_view_type = "NEW_AND_OLD_IMAGES"

  attribute {
    name = "id"
    type = "S"
  }

  tags = {
    Name        = "SlackBot"
    Environment = "dev"
  }
}
</pre>
>
>Now, we are need add to terraform file the resources of kind role, this is required to allow invoke our lambda, add 
>access to dynamo db and cloudwatch to logging
>
````
resource "random_id" "unique_suffix" {
  byte_length = 2
}

resource "aws_iam_role" "lambda_exec" {
  name_prefix = local.app_id

  assume_role_policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": "sts:AssumeRole",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Effect": "Allow",
      "Sid": ""
    }
  ]
}
EOF

}

resource "aws_iam_role_policy_attachment" "lambda_AmazonDynamoDBFullAccess" {
  role       = aws_iam_role.lambda_exec.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess"
}

resource "aws_iam_role_policy_attachment" "lambda_CloudWatchFullAccess" {
  role       = aws_iam_role.lambda_exec.name
  policy_arn = "arn:aws:iam::aws:policy/CloudWatchFullAccess"
}
````

>After that, we are need build our code, dou you need run the follow code(navigate to folder that contain code lambda) 
>,or execute to file sh as it's on github's  
>repository
>
````
rm -rf ./../bin &&\
 mkdir ./../bin &&\
 GOOS=linux go build -o ./../bin/slack-bot-release-aws-lambda &&\
 cd ./../bin &&\
 zip slack-bot-release.zip slack-bot-release-aws-lambda
````

> Then, now we are should create infraestructure on aws using terraform, for it, navigate to folder terraform and
> execute the follows steps:
> - terraform init
> - terraform plan
> - terraform apply --auto-approve=true
>
>If it's ok, only left a step. Let's go create chatbot, because terraform not has a resource to automatize creation 
>the service chatbot, we are should create it manually, for it follow this steps:
> -  You should access to aws chatbot under aws console
>  -  Choose Slack as client
>      -  Choose the workspace in dropdown list
>  -  Configure new channel
>      - On details set any name, e.g. bot-para-notificar-releases
>          - Note: not check logging. it apply charge
>      - Under Channel type, choose Private
>      - Enter channel ID (e.g. T026YK7T0Q5)
>          - if you haven't any channel, you shoud go to slack for create channel
>          - if you have channel for it, enter id. (On my case it's C0271L15MBL)
>      - Permissions
>          -  Set name e.g. bot-para-notificar-releases-role
>          -  Policy templates :
>              -  Read-only command permissions
>              -  Lambda-invoke command permissions
>      - Notifications
>          - select region sns topics(on my case, it's us-east-1)  
>          - select topic  
>  -   Finally, click on configure button

## Testing our bot
>Now, the better moment, let's go test, do you should go to slack and on the channel enter the follow
>snippet and change data if you want
>
<pre>
@aws lambda invoke --function-name slack-release-service-dev
--region us-east-1 
--payload {
    “title”: “titulo”,
    “description”: “se realizo refacto de ...”,
    “product”: “MyProduct”,
    “detail”: “https://any-endpoint.com”,
    “team”: “MyTeam”,
    “status”: “IMPLEMENTADO”,
    “owner”: “Gustavo Martinez”,
    “result”: “EXITOSO”,
    “observations”: “sin observaciones”
}
</pre>

>this return a alert indicating the action to execute, 
>this is for example
>
![Alt text](/images/aws-slack-bot/slack-aws.png)

>After that, it shoud ask if you want run command, for it click on "Yes", then 
![Alt text](/images/aws-slack-bot/aws-confirm.png)
>
>Finally, do you watch the follow message from cloudwatch alert and if you verify
>on dynamo will be new release on table
>
![Alt text](/images/aws-slack-bot/cloudwatch.png)
>For doubt or suggestions write me
>Thank you!!!