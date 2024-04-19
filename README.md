# AWS DevOps Concept

## CodeCommit:
- notification: on codecommit events, send notification to sns or slack
- triggers: on codecommit events, send notification to sns or call lambda func 
- monitor events of codecommit(ex pullrequestcreated) with eventbridge, then lambda,sns,codepipeline
- migrate from github to codecommit: git clone->create repo on codedommit->push to coddecommit
- cross region replication: eventbridge(on referencecreated and referenceupdated->ecs task->clone repo->replicate on dest codecommit on other region)
- approval template: create user pool and choose number of users to approve pull requests

## CodePipeline:
- can be triggered on eventbridge(codecommit events, best option), http webhooks(script), polling(regular github check)
- source artifact: s3,codecommit,ecr,github
- change detection of codecommit ways
	- cloudwatch event(faster): creates an event bridge rule
	- codepipeline(slower): periodically checks
- build,test: codebuild,jenkins
- deploy: codedeploy,cloudformation,ecs,s3,beanstalk
- we can have multiple stages, in each stage we can have multiple action groups(run sequentially), in each action group we can have multiple actions(run in parallel)
- manual approval:triggers sns->email to user. user should have GetPipeline and PutApprovalResault allowed in its iam permission
- cloud formation integration: build app-> create test infra using cf->test ap->delete test infra using cf->deploy to production infra
- cloud formation as target: create,delete,update stack
- if failed: eventbridge->sns->email to user
- can invoke lambda and step func

## CodeBuild
- by default runs outside of our vpc
- codebuild service role: allow codebuild to access aws services(code commit,parameter store,s3(for artifacts),cloudwatch logs,kms)
- can be triggered by eventbridge(on codecommit events) or github events (web hooks)
- report section in buildspec.yml: define path of test resault
- ec2 deployment hooks(in-order): beforeblocktraffic,afterblocktraffic,applicationstop,beforeinstall,afterinstall,applicationstart,validateservice,beforeallowtraffic,afterallowtraffic
	- hooks are scripts
	- blocktraffic and allowtraffic is used only when we have elb
- blue green deployment: need elb, manual mode(create new ec2 instances with new tags),auto(new asg will be created)
- blue instance termination options: terminate,keep alive(deregister from elb)
- deployment configuration: AllAtOnce,HalfAtATime,OnceAtATime, or custom

- deploy on ecs(supports only blue green): codebuild(get code from codecommit->create docker image->push new image on ecr-> create new task definition revision->update appspec.yml->put appspec.yml on s3) -> deploy on ecs cluster(shift traffic modes: Canary, Linear, or All At Once)
- ecs deployment hooks: beforeinstall,afterinstall,afterallowtesttraffic,beforeallowtraffic,afterallowtraffic
- hooks are lambda funcs

- deploy on lambda(supports only blue green): codebuild(create new lambda version->update appspec.yml->put appspec.yml on s3)->codedeploy updates the lambda alias to new version(shift traffic modes: Canary, Linear, or All At Once)
- lambda deployment hooks: beforeallowtraffic,afterallowtesttraffic
- hooks are lambda funcs

## EC2 Image Builder
- can be run on schedule 
- with codepipeline: build app with codecommit->build ami with cloudformation(ec2 image builder)->update ami of asg instances with cloud formation
- aws ram:share images across accounts and regions
- store latest ami id in parameter store: ec2 image builder->sns->lambda->store in parameterstore->use it in cloudformation referencing parameter store

## CloudFormation:
- cloudformation should have enough permissions to create or delete its resources
- by default cloudformation has the same permissions as the user that is running the stack
- but if the user don't have enough permissions:
	- create a role for cloud formation in iam with enough permissions
	- on iam permissions of user add iam:PassRole
	- on creation of stack, select the created role for cloudformation
	- => user can run the stack
- DeletionPolicy for resources: retain(save resource),snapshot(save data),delete(default)
- termination protection: prevent accidental stack deletion
- user data in cf should be passed to !Base64 func
- user data problem: we don't know when it finishes
- cloud formation helper scripts (metadata): cfn-init(package,groups,user,source,files,commands,services),cfn-signal(signal when execution finishes),cfn-get-metadata,cfn-hup(check for metadata change)
- if rollback failed: manually fix error, or skip failed resources
- custom resources: a lambda func that will be run on stack create,update,delete
- usecase: empty s3 bucket before delete stack(because non empty s3 bucket can not be deleted on stack deletion)
- service role: give permission to cloudformation stack to create,update,delete aws resources
- stackset with organization: to auto deploy stack instances to new accounts in organization
- ssm parameters
- we can define parameter in cloudformation and reference it in resources(usecase:ami id)(only latest version will be resolved)
- we can use dynamic parameters directly inside resource definitions which will be resolved from parameterstore or secret manager(usecase: passwords)(we can specify versions)

- cloudformation stackset:
- create a iam role for admin account (administrator role)(use cource resources)
- create a iam role for every account which we want to deploy a stack on it(execution role)(use cource resources)
- create cloudformation stack and select these 2 role



## Beanstalk
- if we create db in beanstalk creation wizard, by deleting beanstalk, db also will be deleted
- => craete db in advande, and select it in beanstalk(select same vpc for db and beanstalk)

- beanstalk update strategies (deployment policy)
- all at once,rolling,immutable,blue green(manually create new env, optionaly redirect some users to new env using route 53, when we are happy swap urls and kill old env),traffic splitting(canary)

- Beanstalk with DB
- create iam role for ec2 with permissions : beanstalkwebtier, beanstalkworkertier, beanstalkmulticontainer
- create beanstalk project and assign the created role to its ec2 instance profile
- it creates the resources using cloudformation under the hood

## Service Catalog
- for joniur workers
- admins write cloud formation templates as products, put them together and create protfolio, create iam permission which tells who can access to this portfolio.
- => joniur employees can create only the predefined products(ex:ec2 with certain configs)
- users should have access to service catalog, all other permissions should be attached to launch constraint(an iam role)(access to cf,services in cf,s3 bucket of cf template)

## SSM
- create new iam role for ec2(allow amazonssmmanagedinstancecore)
- create ec2 instances(attach created iam role,ami=amazon linux 2(because it has ssm agent preinstalled)) and add tags to them, and config security group to allow http traffic if needed
- now we can see ec2 instances are in ssm fleet manager
- in aws resource groups, create ec2 resource groups base on created tags
- in ssm create a document(target type=ec2, enter commands on run commands part of document)
- in ssm in run commands,select the created document and as its target, select instances or created resource group

- ssm session manager: to ssh to ec2 instances, no need to allow ssh and ssh key, control who can ssh to which instance using iam role
- sessoin menager to ssh to instances in private vpc: need 2 eni(for ssm service and ssm session manager)

- ssm patch manager
- run patch baseline (amazon owned or custom) on schedule (maintenance window) on recource group or instances
- on ssm, create maintenance window, 
- on maintenance window config, as run command, select aws-runpatch, then select targets

- dhmc: default host management configuration
- eliminate the need of "create new iam role for ec2(allow amazonssmmanagedinstancecore)" in ssm
- enable it in ssm fleet manager, update ssm agent to version 3.2.582.0(see other pre req). when ec2 instance is created, get the required role

- ssm automation usecase: stop and start and scale instances at certain time(using eventbridge), build ami on schedule (eventbrige(schedule)->automation(create ami)->eventbridge(invoke)->lambda(update parameter store)), remadiate non compliant resources(aws config->automation(change resource))

## Stage Variables
- in api gw, create a resource(ex: /myresource)
- create a lambda func (ex: /myfunc),create v1 and v2 from it,
- create aliases for lambda versions (DEV->$latest,TEST->v2,PROD->v1)
- in api gw, create method for /myresource (point to <lambda_arn>:${stageVariable.lambdaAlias}). it shows a command to add permission to api gw to run lambda func. copy and paste it on cloud shell 3 times replacing ${stageVariable.lambdaAlias} with DEV,TEST,PROD each times (it will add a policy to each alias)
- test the method of /myresource, entering stage variable(lambdaAlias) as DEV,TEST,PROD
- deploy the resource to 3 stages(dev,test,prod), and in their stages variable, add lambdaAlias=DEV or TEST or PROD for each stage
- now <api_gw_url>/prod/myresource will invoke PROD alias

- API gateway canary
- on api gateway deploy a stage that has a resouce with method that invoke v1 of lambda func
- enable canary for the stage and choose the % on stage setting 
- create v2 of lambda func
- edit stage method to invoke v2 of lambda func, deploy it
- it automatically send x% of traffic to v2
- when we are happy, do promote canary

## Kinesis Data Firehose (delivery streams)
- take data from app,sdk,kinesis data stream, cloudwatch and gives it to s3, opensearch, splunk, http endpoint. it optionally can modify data using lambda func.
- it is near real time(every 60 sec or wait untill it becomes over 1MB then sent them)
- ex1: cloudwatch->subscription filter->firehose->s3->athena to analyse logs
- ex2: cloudwatch->subscription filter->firehose->opensearch to create dashboard

- kinesis data stream conumer scaling: monitor IteratorAgeMilisecons on cloudwatch metrics and create alarm based on that to scale out consumers

## Eventbridge Input Transformation
- we can change the format of event that is sent to (or from) eventbridge

## S3 Event Notifications
- if on s3 events (read,write,...) we want to send it on sns,sqs, or invoke lambda func, we should allow s3 on resource policy of destination (sns,sqs,lambda) to write or invoke

## Cloudwatch Anomaly Detection: use ML
- lookout for metrics: detect root of anomalies with ml

## Health Dashboard: 
- monitor health of aws services(from problem of aws)
- service(for all aws services), your account(for services related to my account)
- health event notification: using eventbridge

## EC2 Status Check
- on ec2-> status check-> action-> create alarm->
	- select sns topic
	- alarm action=recover/reboot/stop/terminate
	- sample=status check failed: either/instance/system
- select recover for status check failed: system (when aws hardware faces problem, recover instance(terminate it and start it on other hardware))
- select reboot for status check failed: instance (when instance faces problem(software or network), reboot instance(solve software problems))

## DLQ for SNS
- we can define 1 dlq (and dlq redrive policy) per subscription, not topic (ex:1 for http, not specific email)

## AWS config
- to check compliance of our resources(if they have desired configuration) based on some rules, ex all instances should have restricted ssh access in their security group
- config can not prevent hapenning of a config
- but for non compliant resources can trigger remediation action (ssm automation document)
- if found non compliant resource: aws config->eventbridge->sns
- check can be done on schedule or on any config change
- aggregator: aggregate config of different regions and accounts to a central account

- setup config, select resources that you want to get monitored, select s3 bucket,select rules to make a resource compliant or non compliant,add remediation action

- configuration recorder
- record config items(each changes on configuration of aws resources), [enable it in all accounts,prevent all accounts to disable it using scp policy, create rules on all accounts] using cloudformation stack set, using aggregator aggregate all config items from all accounts and regions, and send it to the destination account (if don't use organization, all aggregated accounts should be authorized in aggregator)

- conformance pack
- define resources and rules and remediations in yaml format-> codecommit-> codebuild-> deploy using cloudformation

## Organization
- to manage multiple accounts at the same time

## AWS Control Tower
- create landing zone and in its config select monitored regions,select ou s, create account for log, archive,enable cloudtrail,

## EKS logs
- control plane to cloudwatch, pods log to cloudwatch(using fluentd or fluent bit),pods metric to cloudwatch(using cloudwatch agent)

## DMS
- database migration service, migrate db to aws, source and destination db engines could be different(convert using sct(schema convertion tool)), should be run on ec2
- can be used to syncronous replication

## Storage Gateway
- to have hybrid cloud data storage(expose s3), needs a file gateway on on-premis database connected to storage gw

## ASG lifecycle hooks
- script to run before making an instance in-service and before instance termination
- use eventbridge to call lambda func on temination and creation of instances on asg
- warm pool: on scale out instances may need some time to get ready, is used to reduce this delay, instances in warm pool can be in running or stoped or hibernated states
- warm pool instance reuse policy: after scale in, terminate instance or put it again in warm pool

## ELB Traget Group weighting
- for blue green deployment and canary
- elb dual stack: to support both ipv4 and ipv6

## blue green deployment
1) with alb 
2) with route53 
3) api gw stages 
4) lambda alias

## multi region services
- dynamodb global,config,aurora global, vpc peering,route53,cloudfront

## disaster recovery
- rpo(recovery point objective,last backup), rto(recovery time objective,next available time(successful recovery))
- strategies: backup and restore(high rpo),pilot light(small varsion of app is always running on cloud),
	- warm standby(full system up and running on cloud but at min size, scale in case of failure), hot site/multi site(low rto, full scale up and running on cloud, actice-active)
