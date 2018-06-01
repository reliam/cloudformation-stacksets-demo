## Cloudformation Stack Sets demo

Before we begin, let's establish some metadata:

```
ADMIN_STACK_URL="https://s3.amazonaws.com/cloudformation-stackset-sample-templates-us-east-1/AWSCloudFormationStackSetAdministrationRole.yml"
ADMIN_STACK_NAME="reliam-stackset-admin-demo"
TARGET_STACK_URL="https://s3.amazonaws.com/cloudformation-stackset-sample-templates-us-east-1/AWSCloudFormationStackSetExecutionRole.yml"
TARGET_STACK_NAME="reliam-stackset-execution-demo"
EXAMPLE_TEMPLATE_FILE="example-stackset-template.yml"
EXAMPLE_TEMPLATE_NAME="example-hello-world-lambda"
MANAGEMENT_ACCOUNT="" # 12 digit account id
TARGET_ACCOUNT="" # 12 digit account id
DEFAULT_REGION="us-east-1"
```

These values are referenced down below. If you want to follow along and deploy stack sets via the CLI, copy the readme to a shell script and strip out the readme text.

#### Pick an AWS account to be your "management" account

This account will use an IAM service role to assume a role into other accounts and deploy the stack sets. You will need to know the account number since the execution role will need to reference it to allow the cross account access.

Amazon publishes Cloudformation templates which can be leveraged for the setup and are referenced above in the metadata.

#### Deploy the service role to management account

```
echo -e "[+] Creating Cloudformation Stack in Management account '${MANAGEMENT_ACCOUNT}' for Cloudformation service assume role"
sleep 1
$ aws cloudformation create-stack \
    --template-url "${ADMIN_STACK_URL}" \
    --stack-name "${ADMIN_STACK_NAME}" \
    --region "${DEFAULT_REGION}" \
    --capabilities CAPABILITY_NAMED_IAM
```

#### Pick an AWS account to be a "target" account

This account will hold an IAM role that the management account assumes a role into.

#### Deploy the service role

You can do this via CLI if you have API access, otherwise, go into the console on the target account and launch the above ExecutionRole stack.

```
echo -e "[+] Creating Cloudformation Stack in Target account '${TARGET_ACCOUNT}' for management account Cloudformation service to assume a role"
sleep 1
aws cloudformation create-stack \
    --template-url "${TARGET_STACK_URL}" \
    --stack-name "${TARGET_STACK_NAME}" \
    --region ${DEFAULT_REGION} \
    --capabilities CAPABILITY_NAMED_IAM \
    --profile "awscli_profile_target_account" \
    --parameters ParameterKey=AdministratorAccountId,ParameterValue=${MANAGEMENT_ACCOUNT}
```


#### Create the Stack Set in Management Account

The Stack Set is just a resource that encompasses any number of stacks underneath it. The Stack Set is configured as a single resource with a Cloudformation template applied to it. After the Stack Set is created, you can add additional stacks and target accounts to start deploying outward.

```
echo -e "[+] Creating Cloudformation Stack Set in Management account '${MANAGEMENT_ACCOUNT}' with template '${EXAMPLE_TEMPLATE_FILE}'"
sleep 1
aws cloudformation create-stack-set \
    --template-body "file://${EXAMPLE_TEMPLATE_FILE}" \
    --stack-set-name "${EXAMPLE_TEMPLATE_NAME}" \
    --region "${DEFAULT_REGION}" \
    --capabilities CAPABILITY_NAMED_IAM
```

#### Create a Stack Set Instance in Management Account

Once the stack set is available, you deploy a *stack set instance* to it containing the AWS account ID of each target account you want to deploy resources to. You also can specify multiple regions for the instance to deploy to.

```
echo -e "[+] Creating Cloudformation Stack Instance in Management account '${MANAGEMENT_ACCOUNT}' and deploying resources"
sleep 1
aws cloudformation create-stack-instances \
    --stack-set-name "${EXAMPLE_TEMPLATE_NAME}" \
    --accounts "${TARGET_ACCOUNT}" \
    --regions "us-east-1" "us-east-2" "us-west-1" "us-west-2" \
    --region "${DEFAULT_REGION}"
```

Once this initiated, the accounts and regions specified become available as instances under the stack set. The Cloudformation service will attempt to assume a role into each account with the given IAM role (the default role in this example) and deploy the designated stack set template to each account in each region specified.

This can be used in conjunction with other scripts and tool to be able to programmatically deploy infrastructure en masse.
