# AWS PrivateLink with Snowflake

![architecture](/images/01_privatelink.jpg)

## Background
Some companies that own business-critical data would like to connect with Snowflake using a private connection, so that the traffic remains within the cloud provider's network (AWS/Azure/GCP) and does not traverse the public internet.

In this exercise, I would like to demonstrate my understanding of private connectivity with Snowflake using AWS PrivateLink. I set up an EC2 instance in the same AWS VPC as the private endpoint connecting to Snowflake via PrivateLink. We will access Snowflake from the EC2 CLI.

To ensure the private connectivity with Snowflake, we need to set up 2 interface VPC endpoints.

## Why do we need 2 interface VPC endpoints?

1. One is needed to connect our AWS VPC with the Snowflake VPC via PrivateLink.

2. Another is needed to connect to Snowflake internal stage (a Snowflake-managed S3 bucket) via PrivateLink. Since S3 is not directly linked to any VPC, a separate VPC endpoint is required to establish the private connection.
 
## Highlight Points in This Architecture

1. How to ensure the EC2 instance in the AWS VPC is using PrivateLink to access Snowflake?

	- Update the EC2 instance's security group outbound rules to allow traffic only to the security groups of the 2 VPC endpoints on ports 80 and 443. Ensure that there is no public egress for the EC2 instance.


2. How to block public access to Snowflake for enhanced security?

	- Set up 2 layers of network policies:

		a. Account Level: Block all public IP addresses for the account and allow access only through private endpoints.
	
		b. User Level: Whitelist a few IP addresses for accessing Snowflake via the public route as a backup plan. User-level network policies will override account-level policies, so set up the user-level policies before applying account-level block on public access. 
	
3. How to test the connection between EC2 instance in the AWS VPC and Snowflake?

	- Install `SnowSQL` on the EC2 instance. Then, log in `SnowSQL` using privatelink-url and upload files to Snowflake internal stage

	- Test Case #1: User1 accesses Snowflake via privatelink-url without any IP blocking set up in Snowflake.

	- Test Case #2: User2 accesses Snowflake via privatelink-url with all public IPs blocked, ensuring User2 can no longer access Snowflake using the public internet.

	- Test Case #3: Both users access Snowflake via public URL.


## Steps

I followed the steps outline in the following resources.

- AWS PrivateLink and Snowflake:

  - https://docs.snowflake.com/en/user-guide/admin-security-privatelink
  - https://interworks.com/blog/2023/11/07/configure-aws-privatelink-with-snowflake/
 	

- Snowflake Internal Stage with AWS PrivateLink:

  - https://docs.snowflake.com/en/user-guide/private-internal-stages-aws
  - https://interworks.com/blog/2024/02/06/configure-aws-privatelink-to-securely-connect-to-snowflake-internal-stages/


After reviewing the resources, I created a CloudFormation stack using the following command:

```
aws cloudformation create-stack --stack-name privatelink-stack --template-body file:///Users/path/to/ec2_privatelink_cfn_template_2.yaml --capabilities CAPABILITY_NAMED_IAM
```

Explanation:

- `--stack-name privatelink-stack`: Specifies the name of the CloudFormation stack.
- `--template-body file:///Users/path/to/ec2_privatelink_cfn_template_2.yaml`: Points to the local file containing the CloudFormation template.
- `--capabilities CAPABILITY_NAMED_IAM`: Allows the stack to create IAM resources.


## Testing the Connection Using PrivateLink

### Part 1: Private Connectivity

1. Get the `allowlist.json` from Snowflake. Run the following query to retrieve the allowlist:

```
SELECT SYSTEM$ALLOWLIST_PRIVATELINK();
```


   Since I have created 3 records for private endpoint pointing to Snowflake service and 1 record for private endpoint pointing to Snowflake internal stage in Route 53,
   I would need to edit the `allowlist.json` to match these records.


2. On the EC2 instance, download `snowCD` and use it to test private connectivity with Snowflake.
 
    Objective: Verify that DNS routing is correct and traffic is routed to the private endpoints properly.

  - Upload `snowCD` and `allowlist.json` to EC2 instance.
	```
	scp -i Downloads/keypair.pem  Downloads/snowcd-1.0.5-linux_x86_64.gz ec2-user@1.23.45.67:~/.
	scp -i Downloads/keypair.pem  Downloads/allowlist.json ec2-user@1.23.45.67:~/.
	```
  - Run `snowCD` in EC2 instance to check for connectivity.

	```
	ssh -i keypair.pem ec2-user@1.23.45.67
	gzip -d snowcd-1.0.5-linux_x86_64.gz
	chmod +x snowcd-1.0.5-linux_x86_64
	./snowcd-1.0.5-linux_x86_64 allowlist.json
	```
![snowcd](/images/02_snowcd_test.png)


### Part 2: Upload Files to Snowflake Internal Stage Using SnowSQL via PrivateLink

1. Upload CSV to EC2

```
scp -i Downloads/keypair.pem  Downloads/city_hotel_testing123.csv ec2-user@1.23.45.67:~/.
```

2. Install SnowSQL on EC2
    Initially, allow outbound traffic to the public internet in the EC2 security group to install SnowSQL. 

```
curl -O https://sfc-repo.snowflakecomputing.com/snowsql/bootstrap/1.3/linux_x86_64/snowsql-1.3.1-linux_x86_64.bash
chmod +x snowsql-1.3.1-linux_x86_64.bash
bash snowsql-1.3.1-linux_x86_64.bash
```

*Note: Download the version you need. For more details, see*

*- https://docs.snowflake.com/en/user-guide/snowsql-install-config*

*- https://developers.snowflake.com/snowsql/*

3. Update EC2 Security Group Outbound Rules

   Update the EC2 security group outbound rules to allow traffic only to the security groups of the two VPC endpoints (one for the Snowflake service and one for the internal stage) on ports 80 and 443. Ensure there is no egress to the public internet.

4. Use `SnowSQL` to upload CSV to Snowflake internal stage

- **Test case #1**:
  
  For user1, No network policy set up in Snowflake. (user1 can also access Snowflake via public internet)
  
```
snowsql -a <account_locator>.ap-northeast-2.privatelink -u user1
put file://city_hotel_testing123.csv @~;
```
![snowsql1](/images/03_user1_snowsql_privatelink.png)

*=> Result: SUCCESSLLY logged in and uploaded file to Snowflake internal stage via PrivateLink.*

- **Test case #2**:
  
  for user2, Network policy is set up in Snowflake to block all public IPs and only allow private endpoint `vpce-xxxxxx`

```
snowsql -a <account_locator>.ap-northeast-2.privatelink -u user2
put file://city_hotel_testing123.csv @~;
```

![snowsql1](/images/05_user2_snowsql.png)

*=> Result: SUCCESSLLY logged in and uploaded file to Snowflake internal stage via PrivateLink.*

*Note: Set up the network policy for user2 in Snowflake as follows:*

```
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
```

 After setting the network policy to block all public IPs, the `test_user` (user2) will no longer be able to access Snowflake using the public URL.
 

![snowsight](/images/04_user2_snowsight_publiclink.png)


- **Test case #3**:

  Test the public route

  Since the EC2 outbound rules are set to route traffic only to private endpoints, both users will fail to connect via the public route. This is expected behavior given the security configuration.

```
snowsql -a <account_name> -u user1

snowsql -a <account_name> -u user2
```
![snowsql_public1](/images/06_user1_snowsql_publiclink.png)
![snowsql_public2](/images/07_user2_snowsql_publiclink.png)

*=> Result: FAILED to logged in to Snowflake service with timeout error.*

5. Clean up the stack

```
aws cloudformation delete-stack --stack-name privatelink-stack
```

### Problem(s)
**Issue**: I failed to upload file to Snowflake internal stage using private endpoint (PrivateLink), encounting timeout error.

**Solution**: As a prerequisite, I need to enable PrivateLink support of internal stage in Snowflake. Execute the following commands:

```
USE ROLE ACCOUNTADMIN;
ALTER ACCOUNT SET ENABLE_INTERNAL_STAGES_PRIVATELINK = TRUE;

```

## Extension of the Exercise
Set up cross-region VPC peering between the VPC with PrivateLink and another VPC located in a different region. This will demonstrate how to enable private communication with Snowflake when connecting from a cross-region VPC.

This will be addressed as Part 2 of the exercise.


## Reference(s)

- Troubleshooting of AWS PrivateLink with Snowflake
  
https://community.snowflake.com/s/article/AWS-PrivateLink-and-Snowflake-detailed-troubleshooting-Guide

