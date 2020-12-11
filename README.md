# Batch Fargate via CloudFormation

This year there was an announcement in re:Invent that caught my attention: 
[Serverless AWS Batch](https://aws.amazon.com/blogs/aws/new-fully-serverless-batch-computing-with-aws-batch-support-for-aws-fargate/).

I had looked at Batch some time before, and it had always been in my head. My vision was that 
if you had a heavy Batch process, the effort of setting AWS Batch up for small tasks was a tad too much for 
my taste. You could resort to Lambda for short-lived tasks.

These days a use case for the new Serverless Fargate came up. Since the service was announced as GA, and
I saw in the CloudFormation User Guide it was supported, I went ahead to try it (I really frown upon services 
that aren't CloudFormation friendly up to a point where I avoid them, since that lack of support brings me
into a situation where you have to develop manuals for getting reproducibility and auditability, which I 
really consider to be a must in the Cloud).

The following is my experience in getting everything to work. Note that this was **not** trivial for reasons
you will see below.

Reading the article, and looking to see if everything is supported in CloudFormation we know that we have to
create three components: Compute Environment, Job Queue and Job Definition:

## Create a Compute Environment 

From the [launch article](https://aws.amazon.com/blogs/aws/new-fully-serverless-batch-computing-with-aws-batch-support-for-aws-fargate/).

```!json
{
    "computeEnvironmentName": "FargateComputeEnvironment",
    "type": "MANAGED",
    "state": "ENABLED",
    "computeResources": {
        "type": "FARGATE", # or FARGATE_SPOT
        "maxvCpus": 40,
        "subnets": [
             "subnet-xxxxxxxx","subnet-xxxxxxxx","subnet-xxxxxxxx"
        ],
        "securityGroupIds": ["sg-xxxxxxxxxxxxxxxx"],
        "tags": {
            "KeyName": "fargate"
        }
    },
    "serviceRole": "arn:aws:iam::xxxxxxxxxxxx:role/service-role/AWSBatchServiceRole"
}
```

Looking at the [AWS::Batch::ComputeEnvironment](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-batch-computeenvironment.html) object:

 - `Type: MANAGED` is [supported](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-batch-computeenvironment.html#cfn-batch-computeenvironment-type)
 - `ComputeResources.Type: FARGATE` is [supported](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-batch-computeenvironment-computeresources.html#cfn-batch-computeenvironment-computeresources-type)

So my corresponding CloudFormation template fragment is:
```
  ComputeEnvironment:
    Type: AWS::Batch::ComputeEnvironment
    Properties: 
      Type: MANAGED
      State: ENABLED
      ServiceRole: !Ref BatchRole
      ComputeResources: 
        Type: FARGATE
        MaxvCpus: 1
        Subnets:
        - !Ref Subnet1
        - !Ref Subnet2
        SecurityGroupIds:
        - !Ref BatchSG
```

This works correctly :)

## Create a Job Queue

The [launch article](https://aws.amazon.com/blogs/aws/new-fully-serverless-batch-computing-with-aws-batch-support-for-aws-fargate/) references:

```!json
{
  "jobQueueName": "FargateJobQueue",
  "state": "ENABLED",
  "priority": 1,
  "computeEnvironmentOrder": [
    {
      "order": 1,
      "computeEnvironment": "FargateComputeEnvironment"
    }
  ]
}
```

Looking at the [AWS::Batch::JobQueue](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-batch-jobqueue.html) object, this gets
translated easily:

```
  Queue:
    Type: AWS::Batch::JobQueue
    Properties:
      ComputeEnvironmentOrder: 
        - ComputeEnvironment: !Ref ComputeEnvironment
          Order: 1
      Priority: 1
      State: ENABLED
```

This also works correctly! :)

## Create a Job Definition

The [launch article](https://aws.amazon.com/blogs/aws/new-fully-serverless-batch-computing-with-aws-batch-support-for-aws-fargate/) has a sample Job Definition. I
have to advise you that this is where the *fun* starts!

```!json
{
    "jobDefinitionName": "FargateJobDefinition",
    "type": "container",
    "propagateTags": true,
     "containerProperties": {
        "image": "xxxxxxxxxxxx.dkr.ecr.us-east-1.amazonaws.com/test:latest",
        "networkConfiguration": {
            "assignPublicIp": "ENABLED"
        },
        "fargatePlatformConfiguration": {
            "platformVersion": "LATEST"
        },
        "resourceRequirements": [
            {
                "value": "0.25",
                "type": "VCPU"
            },
            {
                "value": "512",
                "type": "MEMORY"
            }
        ],
        "jobRoleArn": "arn:aws:iam::xxxxxxxxxxxx:role/ecsTaskExecutionRole",
        "executionRoleArn":"arn:aws:iam::xxxxxxxxxxxx:role/ecsTaskExecutionRole",
        "logConfiguration": {
            "logDriver": "awslogs",
            "options": {
            "awslogs-group": "/ecs/sleepenv",
            "awslogs-region": "us-east-1",
            "awslogs-stream-prefix": "ecs"
            }
        }
     },
   "platformCapabilities": [
        "FARGATE"
    ],
    "tags": {
    "Service": "Batch",
    "Name": "JobDefinitionTag",
    "Expected": "MergeTag"
    }
}
```

Seeing that [AWS::Batch::JobDefinition](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-batch-jobdefinition.html) resource has explicit mentions
of Fargate:

 - `Type: container`: [supported](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-batch-jobdefinition.html#cfn-batch-jobdefinition-type)
 - `ContainerProperties`: [supported](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-batch-jobdefinition-containerproperties.html)
 - `ResourceRequirements`: [supported](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-batch-jobdefinition-resourcerequirement.html) with special mentions for Fargate
 - `FargatePlatformConfiguration`: not in the resource, but mayeb CloudFormation will just use the latest
 - `PlatformCapabilities`: not in the resource, but maybe CloudFormation will take an educated guess, since the ResourceRequirements states

Note that the ResourceRequirements in the manual specify 0.5, 1, 2, 3, and in the launch we see '512'. This will be important later.

And here I start getting errors:

```
An error occurred (ClientException) when calling the RegisterJobDefinition operation: 
Error executing request, Exception : networkConfiguration not applicable for EC2.,
RequestId: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

So it looks like CloudFormation is creating a JobDefinition for EC2 (and not Fargate!). 

Taking out the NetworkConfiguration part of my CloudFormation I start getting new errors.

```
An error occurred (ClientException) when calling the RegisterJobDefinition operation: 
Error executing request, Exception : Memory must be at least 4 Mib, got 1 Mib.,
RequestId: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

Why is this so? I had read the [ResourceRequirements documentation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-batch-jobdefinition-resourcerequirement.html) 
where it was explicity saying I set decimal values (1 for 1GB of RAM) when using Fargate.

> For jobs running on Fargate resources, then value is the hard limit (in GiB), represented in decimal form, and must match one of the supported values (0.5 and whole numbers between 1 and 30, inclusive) and the VCPU values must be one of the values supported for that memory value

```
An error occurred (ClientException) when calling the RegisterJobDefinition operation: 
Error executing request, Exception : Fargate resource requirements (0.50 vCPU, 1 MiB) not valid., 
RequestId: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

Something here is fishy, because in the launch article the MEMORY parameter is '512'...

I think I'll approach the problem from another angle. I'll try to hand-configure a JobDefinition on the console. 
Lots of tests on the AWS console (which also has some quircky behaviors when doing Fargate Batch) get me convinced that
everything **should** work.

Then a breakthrough. Out of desperation I add two undocumented properties to the JobDescription: `FargatePlatformConfiguration` and `PlatformCapabilites`.

I get an error for `Encountered unsupported properties in {/}: [FargatePlatformConfiguration]`. But not for `PlatformCapabilites`!

After that I decide to try with that fishy MEMORY value that was in the article: BINGO!

So after lots of pain, here we have **a working** Batch Fargate CloudFormation resource:

```
  JobDefinition:
    Type: AWS::Batch::JobDefinition
    Properties:
      Type: container
      JobDefinitionName: { Ref: "AWS::StackName" }
      PlatformCapabilities:
      - FARGATE
      Timeout:
        AttemptDurationSeconds: 600
      RetryStrategy:
        Attempts: 1
      ContainerProperties:
        Command:
        - 'echo'
        - 'hello'
        - 'world'
        Image: 'debian:latest'
        NetworkConfiguration:
          AssignPublicIp: ENABLED
        ResourceRequirements:
        - Type: VCPU
          Value: 0.5
        - Type: MEMORY
          Value: 1024
        ExecutionRoleArn: !GetAtt ExecutionRole.Arn
        LogConfiguration:
          LogDriver: awslogs
          Options:
            "awslogs-group": !Ref LogGroup
            "awslogs-stream-prefix": "deploy"
```

Note that I have also eliminated the `"awslogs-region": "us-east-1"` option from the LogConfiguration Options, since the 
console didn't show that option existed, so I decided to go without it. Later I found all the configuration options 
[https://docs.aws.amazon.com/batch/latest/userguide/using_awslogs.html](here).


# Conclusions

You may have a hard time getting Batch running Fargate with CloudFormation. I hope I've saved you the pain of going through
all the guessing, time and pain. Here you have a [batch-fagate.yaml](complete template). There are a couple of unresolved 
references that you need to adapt to your use case.

As for AWS:
 - The CloudFormation documentation for Fargate Batch is clearly lacking
   - The PlatformCapabilites property which seems to be needed to get a Fargate Job Definition is not documented
   - The ResourceRequirements documentation seems incorrect

# Author, Copyright and License

This article was authored by Jose Luis Martinez Torres.

This article is (c) 2020 Jose Luis Martinez Torres, Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).

The canonical, up-to-date source is [GitHub](https://github.com/pplu/cfn-fargate-batch). Feel free to contribute back.
