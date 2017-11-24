+++
date = "2016-02-06T14:46:39Z"
title = "AWS Shared Account Automation"
description = "Versioning game data in the VCS effectively, even for large amounts of data"

tags = ["go", "aws"]
categories = ["services", "maintenance"]
+++

## The Multi-Account Organization
There are a lot of properties of having multiple AWS accounts in an organization to recommend it as a practice:
Separate invoices for different teams

In some organizations cost allocation is a thing, and being able to get different invoices for different product groups is good. You can achieve some of this using Tags, but various services have some costs that aren’t easily separated.
Security

It can be difficult to avoid giving developers IAM access, since there are an ever increasing number of services and uses for it with assumed roles for EC2 and AWS Lambda.

It can be quite difficult to regulate so that users with some IAM access can use these things but not elevate their own permissions, and can make it more difficult for developers to contribute to having well-locked down roles.

## Partitioning
With per-team accounts, you don’t have to worry nearly as much about one team messing with something critical to another by accident. In an organization with teams of varying experience and services of different levels of critical , this

### Automation Across Accounts
Once you have a load of accounts, getting an overview becomes a challenge for your administrators. Luckily, using trust relationships between accounts mean that you don’t need to create accounts for your administrators on every account you have.

To make your security team happier, I absolutely recommend the advice that you require MFA for any cross-account login.
So now you have a set of administrators on a root account (possibly your consolidated billing account), logging into other accounts easily.
### Automation is everything
Automation of as much as possible is key to using AWS effectively, but we’ve created a problem — our administrators have API credentials and now need to programatically use MFA, and account swapping to do things on other accounts, like maintaining shared roles, checking and updating SSL certificates etc. Using MFA and STS together is not obvious!

It’s also just worth noting that the AWS CLI is not well set up for this usage, and some tools are not well exposed in the console at the moment. As an alternative, it is possible to create service users for specific tasks on each account, but the overhead of that method grows quickly.
### The Flow
Here’s the sequence of things that need to happen:

1. With the administrator’s access key and secret access key, you need to create an STS Session Token, giving the MFA id and a token value from the device. This will return a new key, secret key and session token.
2. Using the new details, you now need to call STS AssumeRole for the required role on each account. You do not give your MFA / token codes this time. This will return a new key, secret key and session token for each account.
3. You now use these last keys, secret keys and tokens as your authorization on the accounts.
It’s important that you create a session token in step one, as the MFA code is fairly time-limited and will expire, but it can also only be used once. If you´re iterating over multiple accounts, you really should get the token first.

### In Code
I’ve chosen to illustrate this in Golang, but it should be relatively understandable and simple to translate to whatever language you like due to the similarities of the AWS libraries.
{{< highlight go>}}
stsSvc := sts.New(session.New())

// assuming a virtual MFA device
serial := fmt.Sprintf(“arn:aws:iam::%v:mfa/%v”, rootAccount, user)
t := &sts.GetSessionTokenInput{
    DurationSeconds: aws.Int64(900),
    SerialNumber: aws.String(serial),
    TokenCode: aws.String(mfa),
}

token, err := stsSvc.GetSessionToken(t)
if err != nil {
    fmt.Println(“Could not get session token using MFA:”, err)
    os.Exit(1)
}

roleSwapConfig := aws.NewConfig().WithCredentials(
    credentials.NewStaticCredentials(
        *token.Credentials.AccessKeyId,
        *token.Credentials.SecretAccessKey,
        *token.Credentials.SessionToken,
    ),
)

roleSwapSession := session.New(roleSwapConfig)

// Now we have a session config that uses a session token generated
// with our MFA code, we can use that repeatedly to assume a
// role that requires MFA:
roleArn := fmt.Sprintf(“arn:aws:iam::%v:role/%v”, account, adminrole)
fmt.Println(“Assuming role on account”, account)

svc := sts.New(roleSwapSession)
uniqueSessionName := fmt.Sprintf(“%d”, time.Now().UTC().UnixNano())

params := &sts.AssumeRoleInput{
    RoleArn: aws.String(roleArn),
    RoleSessionName: aws.String(uniqueSessionName),
}

assumeRoleResult, assumeRoleErr := svc.AssumeRole(params)
if assumeRoleErr != nil {
    fmt.Println(“Unable to assume role”, assumeRoleErr)
    return
}
{{< /highlight >}}
## Uses

Using MFA means that the duration of access is limited — it’s not great for service accounts that are automatically scanning for things continuously and long-term (making a limited read-only shared role, disabling the MFA requirement and rotating keys regularly seems better), but it’s great for any command line tools, especially if they can use a non-MFA read-only role and then elevate themselves to another role that requires MFA for making changes.

In particular, since operations and security departments may not want developers handling and managing SSL certificates themselves, tools for managing certificates across multiple accounts could be highly useful.

Another great use is as a way of pushing out consistent roles for shared non-administrator access across a number of accounts.
