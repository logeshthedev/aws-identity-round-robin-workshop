# Permissions boundaries workshop - Advanced - <small> Build phase </small>

## Setup Instructions

### To setup your environment **expand one of the following drop-downs**:

??? info "Click here if *AWS provided an account to you* (usually at an AWS event)"

	### Step 1: Login to the console and run the CloudFormation template
	
	**Console Login:** Your team should have been given a piece of paper with a URL and credentials. This will allow you to login using AWS SSO. 

	After you login click **AWS Account** box, then click on the Account ID displayed below that (the red box in the image.) You should see a link below that for **Management console**. Click on that and you will be taken the AWS console. Make sure the region is set to US East 2 (Ohio).

	** Click the *Deploy to AWS* button below to launch the CloudFormation stck. **
	
	Region| Deploy
	------|-----
	US East 2 (Ohio) | <a href="https://console.aws.amazon.com/cloudformation/home?region=us-east-2#/stacks/new?stackName=Perm-Bound-Adv&templateURL=https://s3-us-west-2.amazonaws.com/sa-security-specialist-workshops-us-west-2/identity-workshop/permissionboundary/identity-workshop-web-admins-advanced.yaml" target="_blank">![Deploy in us-east-2](./images/deploy-to-aws.png)</a>
	
	1. Click **Next** on the **Select Template** section.
	2. Click **Next** on the **Specify Details** section (the stack name will be already filled - you can change it or leave it as is)
	3. Click **Next** on the **Options** section.
	4. Finally, acknowledge that the template will create IAM roles under **Capabilities** and click **Create**.
	5. This will bring you back to the CloudFormation console. You can refresh the page to see the stack starting to create. Before moving on, make sure the stack shows **CREATE_COMPLETE**.
	
	### Step 2 : Connect to the AWS Cloud9 IDE
	
	1. Navigate to the <a href="https://us-east-2.console.aws.amazon.com/cloud9/home" target="_blank">AWS Cloud9</a> console.
	2. Click on **Open IDE** in the `workshop-environment` under **Your environments**
	3. Click the gear icon in the upper right hand corner to open the Cloud9 **Preferences**. Scroll down to **AWS SETTINGS** and click the button next to **AWS Managed Temporary Credentials** to disable this.
	4. Now go back to the AWS SSO tab (this should be the first tab you opened for the workshop). Click the **command line or programmatic access**. Click the section under **Option 2** to copy the credentials.
	4. Go back to the Cloud9 environment. Type `aws configure` hit enter. Hit enter until you get to the choice **Default region name** and type in `us-east-2`
	5. Then create a file in the `~/.aws` directory named `credentials` and paste in the credentials you copied from the SSO login page. Rename the profile to `default` (it will be named something similar to **Account_ID_AdministratorAccess**)
	4. Now when you run commands from within the Cloud9 IDE the temporary credentials from AWS SSO will be used. 
	4. Move on to **Task 1**.

??? info "Click here if you are *using your own AWS account* (whether you are at an AWS event, a separate event or online on your own)"

	Log in to your account however you would normally. You should use an IAM user or role with admin rights. 

	**CloudFormation:** Launch the CloudFormation stack below to setup the environment:

	Region| Deploy
	------|-----
	US East 2 (Ohio) | <a href="https://console.aws.amazon.com/cloudformation/home?region=us-east-2#/stacks/new?stackName=Perm-Bound-Adv&templateURL=https://s3-us-west-2.amazonaws.com/sa-security-specialist-workshops-us-west-2/identity-workshop/permissionboundary/identity-workshop-web-admins-advanced.yaml" target="_blank">![Deploy in us-east-2](./images/deploy-to-aws.png)</a>

	1. Click the **Deploy to AWS** button above.  This will automatically take you to the console to run the template.  
	2. Click **Next** on the **Select Template** section.
	3. Click **Next** on the **Specify Details** section (the stack name will already be filled - you can change it or leave it as is)
	4. Click **Next** on the **Options** section.
	5. Finally, acknowledge that the template will create IAM roles under **Capabilities** and click **Create**.

	This will bring you back to the CloudFormation console. You can refresh the page to see the stack starting to create. Before moving on, make sure the stack is in a **CREATE_COMPLETE**.

	Move on to **Task 1**.

---

!!! Attention
	### Throughout the workshop, keep in mind where you need to add the Account ID, correctly use pathing and change the region specified if needed (although if you are taking this as part of an AWS event, just use the already specified us-east-2.) Missing any of these items can result in problems and errors like **"An error occurred (MalformedPolicyDocument) when calling the CreatePolicy operation: The policy failed legacy parsing"**.

!!! Tip
	#### Tasks 1, 2 and 3 can be done indepedently if you are working in a team and want to divide up the tasks.

---
	
The three elements of a permissions boundary are represented below. When your team does the **BUILD** tasks in this section you will act as the admins. When your team does the **VERIFY** tasks in the next section you will act as the delegated admins (webadmins).  

![mechanism](./images/permission-boundaries.png)

## Task 1 <small>Create the webadmins role</small>

First you will create an IAM role for the webadmins (Initially this role will trust your own AWS account but in the **Verify** phase you will configure it to trust the other team's account):

* For many of the steps below you will need your account ID. To get that type in `aws sts get-caller-identity'. The account ID will be the first number listed after **Account**.
* Use the following JSON to create a file name trustpolicy1 for the trust (assume role) policy (you can use Nano or your preferred text editor): 
	`{ "Version": "2012-10-17", "Statement": { "Effect": "Allow", "Principal": { "AWS": "arn:aws:iam::Account_ID:root"}, "Action": "sts:AssumeRole" } }`
* Create the webadmin role:
	`aws iam create-role --role-name webadmins --assume-role-policy-document file://trustpolicy1`
* Add the Lambda Read Only policy to the role
	`aws iam attach-role-policy --policy-arn arn:aws:iam::aws:policy/AWSLambdaReadOnlyAccess  --role-name webadmins`


## Task 2 <small>Create the permissions boundary for the webadmins</small>

Next you will create the policy that will be used as the permissions boundary.  The permissions boundary should only allow the following actions: Create log groups, create log streams, put logs and list the files in the webadmins folder of the bucket that starts with `"shared-logging-"` and ends in `"-data"`:

* Use the following JSON to create a file named boundarydoc for the policy document:
	
``` json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "CreateLogGroup",
            "Effect": "Allow",
            "Action": "logs:CreateLogGroup",
            "Resource": "arn:aws:logs:us-east-2:ACCOUNT_ID:*"
        },
        {
            "Sid": "CreateLogStreamandEvents",
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "arn:aws:logs:us-east-2:ACCOUNT_ID:log-group:/aws/lambda/*:*"
        },
        {
            "Sid": "AllowedS3GetObject",
            "Effect": "Allow",
            "Action": [
                "s3:List*"
            ],
            "Resource": "arn:aws:s3:::shared-logging-ACCOUNT_ID-us-east-2-data",
             "Condition": {
                "StringEquals": {
                    "s3:prefix": "webadmins"
             		}
        		}
        }		
    ]
}
```
* Create the policy:
	* `aws iam create-policy --policy-name webadminspermissionsboundary --policy-document file://boundarydoc`

!!! question
	* What will the webadmins attach the permissions boundary to?
	* How does a permissions boundary differ from a standard IAM policy?
	* How could you test the permissions boundary at this point?

## Task 3 <small>Create the permission policy for the webadmins</small>

!!! hint 
	**IMPORTANT!!!!** - The question marks **`????`** in the policy below should be replaced with something that could act as part of a resource restriction.  The end result is that you will have a pathing requirement for the roles and policies. Replacing the **`????`** is one of the key challenges in this workshop and probably the most confusing part. Talk to a faciliator if you have issues with this. 

Next you will create the policy that will be attached to the webadmins role.

* Use the following JSON to create a file named policydoc for the policy document:

``` json
{
    "Version": "2012-10-17",
    "Statement": [
		{
            "Sid": "CreateCustomerManagedPolicies",
            "Effect": "Allow",
            "Action": [
                "iam:CreatePolicy",
                "iam:DeletePolicy",
                "iam:CreatePolicyVersion",
                "iam:DeletePolicyVersion",
                "iam:SetDefaultPolicyVersion"
            ],
            "Resource": "arn:aws:iam::ACCOUNT_ID:policy/webadmins/????"
        },
        {
        	  "Sid": "RoleandPolicyActionswithnoPermissionBoundarySupport",
            "Effect": "Allow",
            "Action": [
            		"iam:UpdateRole",
                	"iam:DeleteRole"
            ],
            "Resource": [
                "arn:aws:iam::ACCOUNT_ID:role/webadmins/????"
            ]
        },
        {
            "Sid": "CreateRoles",
            "Effect": "Allow",
            "Action": [
                "iam:CreateRole",
                "iam:AttachRolePolicy",
                "iam:DetachRolePolicy"
            ],
            "Resource": [
                "arn:aws:iam::ACCOUNT_ID:role/webadmins/????"
            ],
            "Condition": {"StringEquals": 
                {"iam:PermissionsBoundary": "arn:aws:iam::ACCOUNT_ID:policy/webadminspermissionsboundary"}
            }
        },
        {
            "Sid": "LambdaFullAccess",
            "Effect": "Allow",
            "Action": "lambda:*",
            "Resource": "arn:aws:lambda:us-east-2:ACCOUNT_ID:function:*"
        },
        {
            "Sid": "PassRoletoLambda",
            "Effect": "Allow",
            "Action": "iam:PassRole",
            "Resource": "arn:aws:iam::ACCOUNT_ID:role/webadmins/????",
            "Condition": {
                "StringLikeIfExists": {
                    "iam:PassedToService": "lambda.amazonaws.com"
                }
            }
        },
        {
            "Sid": "AdditionalPermissionsforLambda",
            "Effect": "Allow",
            "Action": 	["kms:ListAliases", "logs:Describe*", "logs:ListTagsLogGroup", "logs:FilterLogEvents", "logs:GetLogEvents"],
            "Resource": "*"
        },
        {
            "Sid": "DenyPermissionBoundaryandPolicyDeleteModify",
            "Effect": "Deny",
            "Action": [
                "iam:CreatePolicyVersion",
                "iam:DeletePolicy",
                "iam:DeletePolicyVersion",
                "iam:SetDefaultPolicyVersion"
            ],
            "Resource": [
                "arn:aws:iam::ACCOUNT_ID:policy/webadminspermissionsboundary",
                "arn:aws:iam::ACCOUNT_ID:policy/webadminspermissionpolicy"
            ]
        },
        {
            "Sid": "DenyRolePermissionBoundaryDelete",
            "Effect": "Deny",
            "Action": "iam:DeleteRolePermissionsBoundary",
            "Resource": "*"
        }
    ]
}
```
 
* Create the policy named `webadminspermissionpolicy`:
	* `aws iam create-policy --policy-name webadminspermissionpolicy --policy-document file://policydoc`
* Attach the policy to the webadmins role
	* `aws iam attach-role-policy --role-name webadmins --policy-arn arn:aws:iam::Account_ID:policy/webadminspermissionpolicy`
* When you are done the **webadmins** role should have these two policies attached: webadminspermissionpolicy & AWSLambdaReadOnlyAccess.

!!! question

	* Why were there some blocks above in the policy that used the permissions boundary condition and some that did not?
	* Why are we using pathing here?
	* There are two ways of doing resource restrictions: naming and pathing. Which option allows you to create policies using both the AWS Console and CLI?
	* Why do we add the Deny for DeletePolicy actions regarding the webadminspermissionsboundary & webadminspermissionpolicy?
	
## Task 4 <small>Test the webadmins permissions</small>
	
It's time to check your work and make sure the webadmins are set up properly. The instructions for doing so are in the **[VERIFY phase](./verify.md)** phase. Go to that phase to check your work before handing this off to another team. 

## Task 5 <small>Gather info needed for the **VERIFY** phase</small>

It is time to pass the work on to another team who will work through the **VERIFY** tasks. If you are doing this workshop online on your own then you can just run through the **VERIFY** phase yourself (skip this task).

If you were given a form to fill out then enter the info and hand it to another team (or send this to the other team using whatever method is easiest.) If you followed the recommended naming conventions you only need to enter your **Account ID** and **Resource Restriction**.
 
* Webadmins role ARN:	arn:aws:iam::`YOUR_ACCOUNT_ID`:role/**webadmins**
* Resource restriction for both the roles and policies: /webadmins/`Resource restriction you used`
* Permissions boundary name: **webadminspermissionsboundary**
* Permission policy name: **webadminspermissionpolicy**


Exchange forms with another team and then update the trust policy of the webadmins roles so the other team can assume the role:

* Use the following JSON to create a file name trustpolicy2 for the trust (assume role) policy (replace `ACCOUNT_ID` with your account ID so you can still test this and the `ACCOUNT_ID_OTHER_TEAM` with the other team's account ID:) 
	* `{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"AWS":["arn:aws:iam::YOUR_ACCOUNT_ID:root","arn:aws:iam::ACCOUNT_ID_FROM_OTHER_TEAM:root"]},"Action":"sts:AssumeRole"}]}`
* Update the trust policy on the webadmins roles so both your team  and the verify team can assume the role
	* `aws iam update-assume-role-policy --role-name webadmins --policy-document file://trustpolicy2`



### <small>[Click here to go to the VERIFY phase](./verify.md)</small>