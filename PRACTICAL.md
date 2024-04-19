# AWS DevOps Practical

## CodeCommit
- create an iam user and generate https git user and pass for it
- create code commit repo
- clone repo using https, enter user pass
- copy code in cloned folder
- git add . -> git push

- or create new branch, push, pull request, merge

- branch security best practice: create iam group for junior devs (inline policy for prevent push to master(find from aws docs))
- add junior devs to this group
- => junior devs can not commit on master directly (only pull request)

- create notification rule on pull request, set sns as topic to send email to senior dev (or slack)

- create triggers to call a lambda func (or sns) on commits or pull requests or...

- create cloudwatch event that on events of codecommit, call lambda func or push sns topic

## CodeBuild
- upload code and buildspec.yml on codecommit
- put needed passwords on ssm parameter store
- create s3 bucket for artifacts and enable encryption
- create codebuild project (source=codecommit,new service role,environment vars=passwords from parameter store,artifact store=the s3 bucket, modify service role to get access to s3 bucket)
- add awsssmreadonlyaccess to permissions of codebuild policy which has just created
- start build

- create cloudwatch event that periodically start the build (target=codebuild project)

- create cloudwatch event that on events of codebuild(ex,build failed), call lambda func or push sns topic

## CodeDeploy
- write app code and appspec.yml
- create an s3 bucket and enable versioning
- push the code + appspec.yml zip to s3 bucket
- create an iam role (usecase=ec2,permission=awss3readonlyaccess)
- create ec2 (role=s3readonly,add codedeploy agent (install it in user data),sec group=allow port80, add some tags on it)
- create application in codedeploy (platform=ec2)
- create an iam role (usecase=codedeploy,permission=awscodedeployrole)
- create deployment group(service role=codedeployrole,env=ec2-instances(or optionaly asg),tags=ec2 tags)
- create cloudwatch alarm (ex cpu load over 80%)
- create deployment (select deploymentgroup,revision=code on s3,rollback config based on the alarm)
- wait to deploy, open ec2 public name

- create codedeploy triggers (or cloudwatch event) on any codedeploy event, to send it to sns topic

- codedeploy on on-premises instances
- config instanses to register on codedeploy(using iam role)(search on aws docs)
- on codedeploy/on-premises instances add tag to instances
- create application in codedeploy (platform=on-premises)
- create deployment group(env=on-premises)
- create deployment

- codedeploy on lambda func
- create application in codedeploy (platform=lambda)
- create a service role(usecase=codedeploy,permission=awscodedeployroleforlambda)
- create before and after allow traffic lamda functions
- create deployment group(service role=codedeployroleforlambda)
- ...

## CodePipeline
- create codecommit and codedeploy
- create a pipeline (new servicerole,new s3 bucket,source=codecommit,use cloudwatch events,test=codebuild,deploy=codedeploy)
- add s3 read and write permissions to service role of each state
- release change

- create cloudwatch event on any codepipeline event, to send it to a target (sns,lambda(to send to slack),..)

- custom job in codepipeline using lambda (ex: do http get, and check if it contains some words)
- create lambda func
- add necessary permission to service role of lambda func (to putjobsuccessresault or putjobfailureresault)(find it in aws docs)
- add new stage in codepipeline and invoke the lambda func

- whole codepipeline can be created using cloudformation (to recreate codepipeline)

## Jenkins Plugins:
- amazon ec2: to automaticaly add or remove ec2 instances as jenkins slaves in case of increase or decreasing on jenkins slaves
- aws codebuild: jenkins sends jobs to be run on aws code build (=> no jenkins slave)
- amazon elastic container service and amazon ec2 container service plugin with autoscaling capabilities: run jenkins slaves on ecs
- aws codepipeline: to use jenkins in code pipeline
- artifact manager on s3: save artifact on s3

## CloudFormation with lambda
- write code, zip it, upload it on s3 bucket
- create stack yaml file defining lambda func in it (insert 'bucket,file,ver' of code zip file on s3 bucket as parameter)
- create stack, select yaml file (it will be uploaded to s3 bucket)
- enter parameters ('bucket,file,ver' of code zip file on s3) and run the stack
- if we want to update the code : 
- update code, upload on s3,
- update stack, choose the stack yaml file, insert new version of code zip file from s3

- cfn-init: for user data (startup commands) instead of entering comands in user data, use modules (convert comands to module and put in metadata of the ec2 resource). on start up, the ec2 will fetch the meta data, and do the job

- cfn-signal: whether the cfn-init passes or fails, the stack creation will be continue. using cfn-signal and wait condition, we make the stack creation wait to get the resault of execution of cfn-init. if failes, the stack creation also failes; if passes, stack creation continues

- cfn-hup: starts a deamon, that periodically check if the metadata (for cfn-init) is changed. if changed, fetch it and redo the job. => if we only update the metadata and update the stack, the ec2 will not be replaced (the metadata will be updated automatically)

## Deploy Lambda using SAM
- sam init: create a folder with sample files
 	template.yml: lambda func config(sam template which will be transformed to cloudformation template)
	- app.py: contains app code
	- requirements.txt: for dependencies
	- events.json: things that will be passed to lambda func
	- test
- sam build: get dependancies and build the app 
- sam local invoke: run lambda func localy passing events.json to it
- sam package: upload cloud formation template on s3 bucket
- sam deploy: deploy lambda func
- for update code
- set deploymentpreference in template (ex:canary), build, deploy, => cloudformation create codedeploy and codedeploy do the lambda update in canary way.

## ECS
- create ecs cluster : 
	- it creates an auto scaling group, ec2 instaces in it,run ecs agents on them as a container (configures it using launch instances of asg),
	- creates vpc, security group, iam role (for ecs agent and get docker images from ecr) 
	- ecs agent register ec2 in ecs cluster
- create task definition : configuration of container (image, port, cpu, ram, env var, ...). it creates a service policy role( if we want our containers connect to aws services(like s3) we have to give permission to it)
	- in logging section, config it to send its logs to cloud watch(check if its role has createlogstream and putlogevents permissions)
- create service: how many cointainer should be run based on task definition.(also scaling rules for service (not underlying ec2 instances))
- link service to alb: to lb requests to services (alb and its securitygroup should be created in advance, and allow traffic from security group of alb on security group of ec2 instances), it creates new target group. source port of service should not be configured(ex 0:80)(=> selected randomly => we can have multi container on same instance, alb can find s port dynamicaly)

- to delete a service: first set desired number of service to 0, then delete the service.

- fargate advantages: scaling service without thinking about underlying ec2 instances

- create cloudwatch events to send to a topic on events of ecs

## OPSWork: 
- for chef cookbook (manage infrastructure and app in layers)
- life cycle of each layer:
	- setup: cookbook which is run after instance startup
	- configure: when an instance enters or leave online state, add eip or attach elb this cookbook will be run on all instances
	- deploy: to deploy app
	- undeploy: delete an app
	- shutdown: before instance termination

## Unified Cloudwatch Agent
- to send logs and metrics from ec2 to cloudwatch
- create ec2 instance, create an iam role with cloudwatchagentserverpolicy and cloudwatchagentadminpolicy permission and attach to it
- ssh to ec2 instance, download cloudwatch agent, install,launch its wizard and select the source logfile and put the config in parameter store(final step of wizard)
- start agent using create config

- use case: send ram metric of ec2 to cloud watch which is not available by default on cloudwatch
	- send logs of web server on cloudwatch, create a filter on that log for 404 status (which creates a metric based on that), create an alarm based on that => if number of 404s reaches to a threshold, we recieve an email

- we can send cloud watch logs to 
	- 	lambda to do some job on it or give it to another aws service
	- 	to kenesis firehose to save it on s3 bucket or splank or elastisearch
	- using subscription filter

## SSM
- (like ansible) to manage ec2 instances using ssm agent
- ssm quick setup: to create a role for ec2 instances(ssmmanagedinstancecore permission) and a role for ssm
- managed instances: add instances to ssm
	- ec2 instances
	- 	create ec2 instances using amazon linux2 ami 
	- 	attach the created iam role to it, add tags to it(ex:env=dev))
	- 	automatically it will be added to ssm managed instances
	- on premises instances
	- 	go to hybrid activation, and create activation code and id
	- 	ssh to on-premises instance, download and install ssm agent, register ssm agent using activation code and id,start agent process
	- 	automatically it will be added to ssm managed instances (add tags to it(ex:env=dev))
- resource group: group managed instances based on instance tags (ex: prod, dev)
- documents: a group of commands or automations
	- like aws-updatessmagent(to update ssm agent), aws-runshellscript(to run commands)
- run command: to run documents on managed instances or resource groups
	- create a run command(document=aws-runshellscript), add commands that you wanna run in it,select target (instance or group or tag), select execution rate, select fail condition,select output(put output of commands on s3)
	- run it(it will be run on all selected targets)
- parameter store: to store key value parameters (string,stringlist,securestring)
	- can be updated and it keeps the version
	- can be get by ec2 instances using ssm get-parameter command (optionaly decrypted)
- patch manager: apply patches to operating systems
	- create run command (doc=aws-runpatch)
	- create patch baseline(how to apply patches and optionaly sellect patch sources) or use default baseline
	- create maintenance window (how often the patch baseline should be applyed)
	- register targets to maintenance window using maintenance window config
	- register patch baseline to maintenance window using maintenance window config(register run command)
	- it will be applied on defined schedule
- inventory: track everything that is running on managed instances
	- setup inventory and setup schedule to collect info,select what should be tracked,select output(s3)
- automation: run multiple tasks sequentialy or add manual approval in it
	- usecase: create custom ami (launch instance,update it,stop instance,create ami)
	- create iam resources using cloud formation stack (look the cource resources)
	- execute automation(document=aws-updatelinuxami,select source ami id, iam resources selected by default)->execute->output: ami id
	- create cloudwatch event (source= ssm automation,target=inspect security(using inspector) or sns topic to change the ami id on parameter store by admin)
- session manager: ssh to managed instances through browser. session history can be saved to s3 or cloudwatch logs

## Config
- track aws resources and their config change and relationship btw resources
	- create a bucket and add needed permissions for config (search on aws doc) to bucket policy
	- create aws config
- rules: check if a resource has specific config or not 
	- ex: if security groups of all ec2 instances allow ssh or not. can be checked on config change or periodically
	- to react to config events, create cloudwatch events, and call a lambda func as its target
	- 	or use aws config remediations
	- create an aggregator to aggregate all configs from all of accounts and regions(need to add 'aws config autorizations' for each account and region)

## Inspector: to identify potential security issues on aws resources
- can be used to network or host assessments
- can be run periodically(it creates a cloudwatch event)
- create ec2 instance, attach created ssm role to it,ami=amazon linux2(already installed ssm agent)
- create inspector and select entered tags on ec2
- preview install inspector agent on ec2 using inspector page (it installes it using ssm agent)
- run assesment template

## AWS Services Health
- to notify when an aws service in a region face some problems
- create a cloudwatch event, source=health, target=sns to send email.

## ASG
- create launch configuration or launch template (ami,ec2 type,iam role,user data,security group)
- optionaly create alb and targetgroup
- create asg:
	- number of instances,targetgroup,health checks using elb
- we can create code deploy and on deploymentgroup select asg as its environment=>the app will be deployed on all existing and new instances of asg (it's better before deploying new version of app with code deploy, suspend launch on asg and remove suspend after deployment(otherwise new instances will have old version))

## HTTPS on ALB
- on route53: create a domain name 
- on cert manager: add domain(or subdomain) and wait until is issued
- on route53: create cname record for cert validation
- on route53: create an alias record (subdomain -> alb dns name)
- on alb: add https listener(action= forward to target group,attach cert)
- on alb: allow https 443 on security groups
- on alb: change http action to redirect to https
