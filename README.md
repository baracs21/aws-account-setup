# Account Setup

In this part, I will describe how you can set up your account securely for private use. If you're working in a corporate environment, you should have no access or the rights to create custom users inside your account.

## Prerequisites

You have already created an aws account, and you have access to your root account. This will be not covered here.

## Doings

Here I will describe step by step what you have to do, to set up your account in a secure way for your private investigations. Alternatively you can configure an access key and secret access key for your root account to access the aws cli programmatically via the cli, but I don't recommend it, to use hard coded credentials for an account, who can do everything inside your account. 

**1. Create your custom admin role to access the aws services via the cli**

- iam > Roles > Create role
- Step 1: Select AWS account and select Require MFA > Next
- Step 2: Add `AdministratorAccess` policy > Next
- Step 3: Enter Role name and optional a description > Create role

**2. Create individual user for your daily work, with minimal permissions**

- iam > Users > Add users
- Enter username and select programmatic access > Next: Permissions
- Select `Create policy` and select JSON. Use the following policy (change ACCOUNT_ID to your account number and ROLE_NAME to the name from the role, you already created in Step 1),:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::ACCOUNT_ID:role/ROLE_NAME"
    }
  ]
}
```
- Select Next: Tags, Next: Review
- Enter Policy Name > Create policy
- Back on the user creation window, add the created policy to the user > Next: Tags, Next: Review and Create user
- Download credentials for later usage > Close
- Select Users again and go to Security credentials.
- Please click on Manage to add an MFA device
- Now you have different options. For simplicity, I chose the Virtual MFA device and a browser plugin.

**3. Change Trust relationship for your custom admin role**

- iam > Roles
- select your created custom admin role and select trust relationships
- Edit the trust policy to the following:
- Update the trusted entities' policy, in the end it should look like, don't forget to add your account id and username:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT_ID:user/USERNAME"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    }
  ]
}
```

**4. Set up your local aws configuration**

- create the following file in ~/.aws/config
```shell
[profile mfa]
region = eu-west-1
output = json

[profile admin]
role_arn = ROLE_ARN (i.e. arn:aws:iam::ACCOUNT_ID:role/custom-admin-role)
mfa_serial = MFA_SERIAL (i.e. arn:aws:iam::ACCOUNT_ID:mfa/mfa_user)
source_profile = mfa
region = eu-west-1
cli_pager =
```
- configure your cli with your previously downloaded credentials, use the source_profile as profile name
```shell
aws configure set aws_access_key_id ACCESS_KEY_ID --profile mfa
aws configure set aws_secret_access_key SECRET_ACCESS_KEY --profile mfa
```

**5. Test your configuration**

To test your application, you have to use the admin profile. With your first usage, you should be prompted to enter your mfa token.

```shell
 aws sns list-topics --profile admin                                   
```

The result should be:
```shell
Enter MFA code for arn:aws:iam::ACCOUNT_ID:mfa/daily-admin: 
{
    "Topics": []
}
```

**6. Export env variable**

- In the end you should export the profile as env variable to avoid entering the profile with every call.
```shell
export AWS_PROFILE=admin
```

## Summary

- You created a role with administrative rights to use all the aws services and hard conditions who can use the role.
- You created a user with minimal permissions and mfa authentication. The only permission the user has, is to assume a specific role.
- You configured your local machine to access aws via the cli.