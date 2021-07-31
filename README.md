# Amazon ECS and the ECS CLI

A simple Flask app is used to walk through the steps of containerizing an app using the ECS CLI.

**Observations**
Following the steps as follows does not produce a setup in which the target group has a registered target. Reason as of yet unknown. As such, the service attempts to deploy tasks, but they fail until it times out and stops producing tasks.

Note that the ECS-CLI is not actively updated these days - https://github.com/aws/amazon-ecs-cli. Instead, emphasis is on ECS Copilot.

## (1) Install the ECS CLI

For Macs, use Homebrew to install the ECS CLI:

```bash
brew install amazon-ecs-cli
```

## (2) Add the App and Dockerfile

Created a simple Flask app, made a requirements.txt file, and wrote a Dockerfile.

## (3) Create the Platform

These templates assume that the VPC, subnets and other networking resources are already deployed.

### Cluster and Container Security Group

Deploy the CloudFormation template that builds an ECS cluster, security groups and an external facing ALB.

```bash
aws cloudformation deploy \
--profile 1s-sandbox \
--stack-name sharina-ecs-cluster \
--template-file cloudformation/ecs-cluster-and-sg.yaml \
--capabilities CAPABILITY_NAMED_IAM
```

### Ensure service linked roles exist for Load Balancers and ECS

"Under most circumstances, you don't need to manually create the service-linked role. For example, when you create a new cluster (for example, with the Amazon ECS first-run experience, the cluster creation wizard, or the AWS CLI or SDKs), or create or update a service in the AWS Management Console, Amazon ECS creates the service-linked role for you, if it does not already exist." [More on AWS](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/using-service-linked-roles.html).

```bash
aws iam get-role --role-name "AWSServiceRoleForECS" --profile [your-profile-name]
```

```bash
aws iam get-role --role-name "AWSServiceRoleForElasticLoadBalancing" --profile 1s-sandbox
```

#### Create the Roles if Needed

```bash
aws iam create-service-linked-role --aws-service-name "ecs.amazonaws.com"
```

```bash
aws iam create-service-linked-role --aws-service-name "elasticloadbalancing.amazonaws.com"
```

## (4) Deploy the App

### Set Environment Variables from the Build

Update the profile name and stack name in the scripts before using these variables. Note that `?OutputKey==` corresponds to the AWS CloudFormation stack --> `Outputs` --> `Key`.

```bash
export clustername=$(aws cloudformation describe-stacks --profile 1s-sandbox --stack-name sharina-ecs-cluster --query 'Stacks[0].Outputs[?OutputKey==`ClusterName`].OutputValue' --output text)

export target_group_arn=$(aws cloudformation describe-stack-resources --profile 1s-sandbox --stack-name sharina-ecs-cluster | jq -r '.[][] | select(.ResourceType=="AWS::ElasticLoadBalancingV2::TargetGroup").PhysicalResourceId')

export vpc=$(aws cloudformation describe-stacks --profile 1s-sandbox --stack-name doug-vpc --query 'Stacks[0].Outputs[?OutputKey==`VpcId`].OutputValue' --output text)

export ecsTaskExecutionRole=$(aws cloudformation describe-stacks --profile 1s-sandbox --stack-name sharina-ecs-cluster --query 'Stacks[0].Outputs[?OutputKey==`ECSTaskExecutionRole`].OutputValue' --output text)

export subnet_1=$(aws cloudformation describe-stacks --profile 1s-sandbox --stack-name doug-vpc --query 'Stacks[0].Outputs[?OutputKey==`PrivateAZASubnetId`].OutputValue' --output text)

export subnet_2=$(aws cloudformation describe-stacks --profile 1s-sandbox --stack-name doug-vpc --query 'Stacks[0].Outputs[?OutputKey==`PrivateAZBSubnetId`].OutputValue' --output text)

export subnet_3=$(aws cloudformation describe-stacks --profile 1s-sandbox --stack-name doug-vpc --query 'Stacks[0].Outputs[?OutputKey==`PrivateAZCSubnetId`].OutputValue' --output text)

export container_security_group=$(aws cloudformation describe-stacks --profile 1s-sandbox --stack-name sharina-ecs-cluster --query 'Stacks[0].Outputs[?OutputKey==`ContainerSecurityGroup`].OutputValue' --output text)
```

### Configure the ECS CLI to Interact with the Cluster

"Configures the AWS Region to use, resource creation prefixes, and the Amazon ECS cluster name to use with the Amazon ECS CLI. Stores a single named cluster configuration in the ~/.ecs/config file." More on the `ecs-cli configure` command [here](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-configure.html).

Update the following flag with a unique name:
`--config-name`

```bash
ecs-cli configure \
--region us-west-2 \
--cluster $clustername \
--default-launch-type FARGATE \
--config-name sharina-ecs
```

Note that there is no need to configure the ecs-cli regarding aws credentials for accounts and such. Just dive into this step and carry on.

After the `ecs-cli configure...` command is run, a `.ecs` folder is created (for Macs) at the root directory.

### Update the ECS Params File

The next step will populate the `ecs-params.yml` file with the environmental variables saved in the CLI. The command uses the `ecs-params.yml.template` file to format those variables when populating the `ecs-params` file.

"When using the ecs-cli compose or ecs-cli compose service commands to manage your Amazon ECS tasks and services, there are certain fields in an Amazon ECS task definition that do not correspond to fields in a Docker compose file... By default, the command looks for an ECS parameters file in the current directory named ecs-params.yml" All about this [here](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-ecsparams.html).

```bash
envsubst < ecs-params.yml.template >ecs-params.yml
```

### Create a docker-compose.yml File

The command that launches the service on the ECS Cluster will look for a docker-compose file.

https://docs.docker.com/compose/

If using the docker-compose file included here, update:

* `ports`
* `awslogs-group`
* `awslogs-stream-prefix`
* `image` - gave unique name... should it be something else?
* `environment`
* service logical id
* `region`



### Launch the Service on the ECS Cluster

Uses a default launch type of Fargate. Note that `ecs-cli compose` will take care of building our private dns namespace for service discovery, and log group in cloudwatch logs, given the flags passed in.

The following command uses the environment variables that were set up in the command line using the `export` command a few steps back.

Note that `up` creates an ECS task definition from your Compose file (if it doesn't already exist) and runs one instance of that task on your cluster.

Update the following flags:
`--project-name`
`--container-name`
`--container-port`
`cluster-config` - this must match the `ecs-cli configure` flag, `--config-name` from a few steps above.

```bash
ecs-cli compose --project-name sharina-ecs-cli service up \
--create-log-groups \
--target-group-arn $target_group_arn \
--private-dns-namespace service \
--enable-service-discovery \
--container-name sharina-ecs-cli \
--container-port 5000 \
--cluster-config sharina-ecs \
--vpc $vpc
```

You will now have three CloudFormation templates deployed - two by amazon from the above command, and the one that you created to deploy the initial infrastructure.

## Check the Status of Things

Update the following flags:
`--project-name` - matches the `project-name` from the above command
`--cluster-config` - matches the `cluster-config` from the above command

```bash
ecs-cli compose \
--project-name sharina-ecs-cli service ps --cluster-config sharina-ecs
```

## Edit Stuff

Note that "After you create a service, you can't change the load balancer name or target group ARN, container name, and container port specified in the service definition."

Remove the service with `ecs-cli compose --project-name sharina-ecs-cli service delete`. This stops the service and deletes it, and removes one of the Amazon created stacks.

## Delete and Cleanup

Remove the service with `ecs-cli compose --project-name sharina-ecs-cli service delete`.

Delete the cluster: `ecs-cli down --cluster <cluster-name>`

