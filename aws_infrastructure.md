# StockClubs AWS Infrastructure
*Authors: Nazar Sopkiv, Alex Sukhanov*  
*Last Updated: January 2022*

# Terraform Code Overview ([each file review](https://github.com/stockclubs-io/stockclubs_aws_2))
### [ostgresql_role](https://github.com/stockclubs-io/stockclubs_aws_2/tree/main/postgresql_role):

- postgresql_role.tf - describes what resources will be created
- providers.tf - connects to AWS account
- variables.tf - required variables which were user in posgresql_role.tf

### [rod_definition](https://github.com/stockclubs-io/stockclubs_aws_2/tree/main/prod_definition):

- stockclubs-api
- stockclubs-mdd
- stockclubs-plaid

*They are just a plug to create the task definitions, etc. But they will be overridden with Backend deploy*

### [3_and_table](https://github.com/stockclubs-io/stockclubs_aws_2/tree/main/s3_and_table):

- s3_and_dynamo.tf - contain the resources which will build s3 bucket and dynamo db table for Pipeline№1
- providers.tf - connects to AWS account
- variables.tf - required variables which were user in s3_and_dynamo.tf

#### Other resources which will be created in Pipeline 3
- acm.tf
- bastion_vpn.tf
- cloudwatch.tf
- ecr.tf
- ecs.tf
- iam.tf
- kms.tf
- load_balancer.tf
- rds.tf
- redis.tf
- route53.tf
- s3.tf
- secret_manager.tf
- sg.tf
- vpc.tf
- waf.tf

#### Use to override variables which are used and scattered over all the above files
- prod.tfvars
- stage.tfvars

### [.github/workflows](https://github.com/stockclubs-io/stockclubs_aws_2/tree/main/.github/workflows) :

- Pipeline_1_s3_and_table - build s3 bucket, so we can save tfstate file off future infrastructure there and dynamo db table to be able activating state lock on tfstate so no one can run terraform apply simultaneously
- Pipeline_2_Plan - plan new upcoming infrastructure or look at what will be changed or deleted after the last changes in terraform files
- Pipeline_3_Apply_full_infrastructure - terraform creates a tfstate in s3 bucket and starts to build/change/destroy resources in specified AWS account and region
- Pipeline_4_grant_postgres_role - runs on our bastion server and creates roles for Postgress (required only if we are going to create DB without any snapshot recovery)
- Pipeline_5_full_destroy - it destroys all the infrastructure which is described in the tfstate file on AWS s3 bucket

## [Stockclubs_backend](https://github.com/stockclubs-io/stockclubs_backend)

##### There are 3 config files that need to be configured with the actual resources on the new infrastructure as: 
- stockclubs_backend/stockclubs_api/stockclubs_api/config.py
- stockclubs_backend/stockclubs_plaid/stockclubs_plaid/config.py
- stockclubs_backend/stockclubs_mdd/stockclubs_mdd/config.py
##### In that files are some values that need to be replaced to the new actual terraform infrastructure in the Production(Stage)Config Block, such as :
>1. AWS_MANUAL_SECRET_NAME, 
>2. AWS_SECRET_NAME,
>3. AWS_REGION, 
>4. SM_REDIS_ENABLE_SSL, 
>5. SM_STOKKER_PLAID_SERVICE, 
>6. SM_UPLOAD_STORAGE, 
>7. SM_STOKKER_LOGGING_FORMAT

### [.github/workflows](https://github.com/stockclubs-io/stockclubs_aws_2/tree/main/.github/workflows) :
- stockclubs_backend_production.yml
- stockclubs_backend_staging.yml

### Use after stockclubs_aws_2 1-3 Pipelines runs. 
- 1-2 jobs will do the testing on the first bastion (dev_testing)
- 3rd job will run on the second bastion (bastion) and do the migrations with the FlyDBMigration service.
- 4th job will build and deploy the ECS service on AWS
- 5th job will send the push notification in Slack


## Deploying the infra in a new AWS Account

#### To deploy the Infrastructure, we need:

- Define basic credentials on [GitHub Secrets](https://github.com/stockclubs-io/stockclubs_aws_2/settings/secrets/actions) :
>1. AWS_SECRET_ACCESS_KEY_STAGE
>2. AWS_ACCESS_KEY_ID_STAGE
>3. AWS_SECRET_ACCESS_KEY_PROD
>4. AWS_ACCESS_KEY_ID_PROD

### To get your access key ID and secret access key
>- Open the IAM console at https://console.aws.amazon.com/iam/.
>- On the navigation menu, choose Users.
>- Select your IAM username (not the checkbox).
>- Open the Security credentials tab, and then select the “Create access key”.
>- To see the new access key, select Show. Your credentials resemble the following:
>- Access key ID: AKIAIOSFODNN7EXAMPLE
>- Secret access key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
>- To download the key pair, choose Download .csv file. Store the .csv file with keys in a secure location.


- Define needed region in [stage.tfvars](https://github.com/stockclubs-io/stockclubs_aws_2/blob/main/stage.tfvars) AND [prod.tfvars](https://github.com/stockclubs-io/stockclubs_aws_2/blob/main/prod.tfvars).
- Create an RDS snapshot for terraform to recover from it. Described in the “RDS from a snapshot” block below.
- Change GitHub secret with new git runner Front & Backend (photo instruction below)
>go  [stockclubs_backend](https://github.com/stockclubs-io/stockclubs_backend/settings/actions/runners/new) , copy token and paste into GitHub secret [BACKEND_ACTION_TOKEN](https://github.com/stockclubs-io/stockclubs_aws_2/settings/secrets/actions)
>go [stockclubs_aws_2](https://github.com/stockclubs-io/stockclubs_aws_2/settings/actions/runners/new) , copy token and paste into GitHub secret [GIT_RUNNER_TOKEN_STAGE](https://github.com/stockclubs-io/stockclubs_aws_2/settings/secrets/actions)


- Run :
>1. [(_PROD) or STAGE_Pipeline_1_s3_and_table](https://github.com/stockclubs-io/stockclubs_aws_2/actions)
>2. [(_PROD) or STAGE_Pipeline_2_Plan](https://github.com/stockclubs-io/stockclubs_aws_2/actions)
>3. [(_PROD) or STAGE_Pipeline_3_Apply_full_infrastructure](https://github.com/stockclubs-io/stockclubs_aws_2/actions)
- After the 1-3 pipelines are done :
>1. Go to AWS Account 
>2. Proceed to EC2 Instances 
>3. Choose the right region on top of page
>4. There will be two instances (bastions) you need to copy their public IPs and paste them into GitHub Secrets 
>5. Dev_deploy IP paste into [DEV_TESTING_HOST](https://github.com/stockclubs-io/stockclubs_backend/settings/secrets/actions) secret in Backend repo
>6. Bastion IP paste into [V2_BASTION_HOST](https://github.com/stockclubs-io/stockclubs_backend/settings/secrets/actions) secret
>7. Add manual secrets in AWS Secret manager —> Manual Secrets
>8. Run the Backend deploy [Pipeline](https://github.com/stockclubs-io/stockclubs_backend/actions)
>9. Use the command with AWS CLI to copy data from the old S3 to the new one

## Giving specific IP addresses access to Bastion machines

- Go to [sg.tf](https://github.com/stockclubs-io/stockclubs_aws_2/blob/main/sg.tf) and look for resource “aws_security_group“ with name “bastion_sg” and add the following block:
`
>===============================================  
>resource "aws_security_group" "bastion_sg" {  
>…   
>  ingress {   
>    description = "Ingress"   
>    from_port   = 22  
>    to_port     = 22  
>    protocol    = "tcp" 
>    cidr_blocks = ["YOUR_IP_ADDRESS"]   
>  }   
>…   
>}     
>===============================================`
## Changing RSA keys for Bastion, ECS and DEV
- If you want to change the SSH key for Bastion 1/2 :
>1. Generate a new ssh key [manually](https://git-scm.com/book/en/v2/Git-on-the-Server-Generating-Your-SSH-Public-Key)
>2. Copy and paste the public key to BASTION_SSH_KEY_STAGE secret in [Github Secrets](https://github.com/stockclubs-io/stockclubs_aws_2/settings/secrets/actions)
>3. Store private key in local machine
>4. Run the [2-3 pipeline](https://github.com/stockclubs-io/stockclubs_aws_2/actions) (stage/prod) so the Bastion will change the old ssh to the new one.

## Changing RDS / Redis Passwords
In order to change the RDS/Redis password, you need to 
visit the GitHub Actions in the [stockclubs_aws_2](https://github.com/stockclubs-io/stockclubs_aws_2/settings/secrets/actions) repo 
change the value in “RDS_PASSWORD_STAGE(_PROD) and REDIS_PASSWORD_STAGE(_PROD) secrets

## Recovering RDS / Redis from a snapshot
- RDS from a snapshot :
>1. Go to aws account N.Virginia region
>2. Search for RDS
>3. Create a snapshot (Make the same name of snapshot as it showed in terraform, in file stage.tfvars, rds_snapshot_name = "migrationv2clustersnapshot")
>4. Move a snapshot to another region
>5. Run Pipeline 2-3
