# AWS PrivateLink with Snowflake

### Background
Some companies that own business-critical data would like to connect with Snowflake using a private connection, so that the traffic remains within the cloud provider's network (AWS/Azure/GCP) and does not traverse the public internet.

In this exercise, I would like to demonstrate my understanding of private connectivity with Snowflake using AWS PrivateLink.

To ensure the private connectivity with Snowflake, we need to set up 2 interface VPC endpoints.

### Why do we need 2 interface VPC endpoints?

1. One is needed to connect our AWS VPC with the Snowflake VPC via PrivateLink.

2. Another is needed to connect to Snowflake internal stage (a Snowflake-managed S3 bucket) via PrivateLink. Since S3 is not directly linked to any VPC, a separate VPC endpoint is required to establish the private connection.
 
### Highlight Points in This Architecture

1. How to ensure the EC2 instance in the AWS VPC is using PrivateLink to access Snowflake?

- Update the EC2 instance's security group outbound rules to allow traffic only to the security groups of the 2 VPC endpoints on ports 80 and 443. Ensure there is no public outbound access for the EC2 instance.


2. How to block public access to Snowflake for enhanced security?

- Set up 2 layers of network policies:

	a. Account Level: Block all public IP addresses for the account and allow access only through private endpoints.
	
	b. User Level: Whitelist a few IP addresses for accessing Snowflake via the public route as a backup plan.
	User-level network policies will override account-level policies, so set up the user-level policies before applying account-level block on public access. 
	
3. How to test the connection between EC2 instance in the AWS VPC and Snowflake?

- Install `SnowSQL` on the EC2 instance. Then, log in SnowSQL using PrivateLink and attempt to upload files to Snowflake internal stage

- Test Case #1: User1 accesses Snowflake via PrivateLink without any IP blocking.

- Test Case #2: User2 accesses Snowflake via PrivateLink with all public IPs blocked, ensuring User2 can no longer access Snowflake using the public internet.

- Test Case #3: Both users attemp to access Snowflake via public URL.


### Problem(s)
Issue: I failed to access Snowflake internal stage from the private endpoint, encounting timeout error.

Solution: As a prerequisite, you need to enable PrivateLink support of internal stage in Snowflake. Execute the following commands:

```
USE ROLE ACCOUNTADMIN;
ALTER ACCOUNT SET ENABLE_INTERNAL_STAGES_PRIVATELINK = TRUE;

```

### Steps

I followed the steps outline in the following resources.

- AWS PrivateLink and Snowflake:

https://interworks.com/blog/2023/11/07/configure-aws-privatelink-with-snowflake/
https://docs.snowflake.com/en/user-guide/admin-security-privatelink

- Snowflake Internal Stage with AWS PrivateLink:

https://interworks.com/blog/2024/02/06/configure-aws-privatelink-to-securely-connect-to-snowflake-internal-stages/
https://docs.snowflake.com/en/user-guide/private-internal-stages-aws

After digesting the documentation, I create Cloudformation stack as below:

aws cloudformation create-stack --stack-name privatelink-stack --template-body file:///Users/path/to/ec2_privatelink_cfn_template_2.yaml --capabilities CAPABILITY_NAMED_IAM

Explanation:

--stack-name privatelink-stack: Specifies the name of the CloudFormation stack.
--template-body file:///Users/path/to/ec2_privatelink_cfn_template_2.yaml: Points to the local file containing the CloudFormation template.
--capabilities CAPABILITY_NAMED_IAM: Allows the stack to create IAM resources.

### Testing the Connection Using PrivateLink

Part 1: Private Connectivity

1. Get the `allowlist.json` from Snowflake. Run the following query to retrieve the allowlist:

```
SELECT SYSTEM$ALLOWLIST_PRIVATELINK();
```

Since I have created 3 records for Snowflake PrivateLink and 1 record for Snowflake internal stage in Route 53,
I will need to edit the `allowlist.json` to match the records.


2. On the EC2 instance, download `snowCD` and use it to test private connectivity with Snowflake:
 
Objective: Verify that DNS routing is correct and traffic is routed to the private endpoints properly.

Commands:

Terminal #1:
scp -i Downloads/keypair.pem  Downloads/snowcd-1.0.5-linux_x86_64.gz ec2-user@1.23.45.67:~/.
scp -i Downloads/keypair.pem  Downloads/allowlist.json ec2-user@1.23.45.67:~/.


Terminal #2:
ssh -i keypair.pem ec2-user@1.23.45.67
gzip -d snowcd-1.0.5-linux_x86_64.gz
chmod +x snowcd-1.0.5-linux_x86_64
./snowcd-1.0.5-linux_x86_64 allowlist.json


Part 2: Upload Files to Internal Stage Using SnowSQL via PrivateLink

1. Upload CSV to EC2

Terminal #1
scp -i Downloads/keypair.pem  Downloads/city_hotel_testing123.csv ec2-user@1.23.45.67:~/.

2. Install SnowSQL on EC2

Terminal #2 (EC2 shell)
curl -O https://sfc-repo.snowflakecomputing.com/snowsql/bootstrap/1.3/linux_x86_64/snowsql-1.3.1-linux_x86_64.bash
chmod +x snowsql-1.3.1-linux_x86_64.bash
bash snowsql-1.3.1-linux_x86_64.bash

(Note: Download the version you need. For more details, see  https://docs.snowflake.com/en/user-guide/snowsql-install-config
https://developers.snowflake.com/snowsql/)

Note: Initially, allow outbound traffic to the public internet in the EC2 security group to install SnowSQL. 

3. Update EC2 Security Group Outbound Rules

Change the EC2 security group outbound rules to route traffic to the security groups of the two VPC endpoints (one for Snowflake service and one for the internal stage) on ports 80 and 443.

4. Upload CSV to Snowflake internal stage via SnowSQL

Test case #1: for user1, No network policy set up in Snowflake

Terminal #2 (EC2 shell)
snowsql -a <account_locator>.ap-northeast-2.privatelink -u user1
put file://city_hotel_testing123.csv @~;

Result: SUCCESSLLY uploaded file to Snowflake internal stage.

Test case #2: for user2, Network policy set up in Snowflake to block all public IPs and allow private endpoint `vpce-xxxxxx`

snowsql -a <account_locator>.ap-northeast-2.privatelink -u user2

Result: SUCCESSLLY uploaded file to Snowflake internal stage.

Note: Set up the network policy for test_user in Snowflake as follows:

CREATE NETWORK RULE block_public_access
  MODE = INGRESS
  TYPE = IPV4
  VALUE_LIST = ('0.0.0.0/0');

CREATE NETWORK RULE allow_vpceid_access
  MODE = INGRESS
  TYPE = AWSVPCEID
  VALUE_LIST = ('vpce-xxxxxx');

CREATE NETWORK POLICY allow_vpceid_block_public_policy
  ALLOWED_NETWORK_RULE_LIST = ('allow_vpceid_access')
  BLOCKED_NETWORK_RULE_LIST = ('block_public_access');

ALTER USER test_user SET NETWORK_POLICY = allow_vpceid_block_public_policy;

Test case #3: Test the public route

Since the EC2 outbound rules are set to route traffic only to private endpoints, both users will fail to connect via the public route. This is expected behavior given the security configuration.

snowsql -a <account_name> -u user1

snowsql -a <account_name> -u user2

Result: FAILED to upload file to Snowflake internal stage.

Note: Both users will fail to connect via the public route due to the EC2 outbound rules restricting traffic to private endpoints only.

5. Clean up the stack

aws cloudformation delete-stack --stack-name privatelink-stack


### Extension of the Exercise
Set up cross-region VPC peering between the VPC with PrivateLink and another VPC located in a different region. This will demonstrate how to enable private communication with Snowflake when connecting from a cross-region VPC.

This will be addressed as Part 2 of the exercise.


### reference(s)

- Troubleshooting of AWS PrivateLink with Snowflake
https://community.snowflake.com/s/article/AWS-PrivateLink-and-Snowflake-detailed-troubleshooting-Guide

