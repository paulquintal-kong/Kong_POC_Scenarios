# How to Integrate AWS Secrets Manager with Kong Insomnia using AWS SSO

## Overview

This guide walks you through authenticating Kong Insomnia with your AWS account using
AWS IAM Identity Center (SSO), creating a secret in AWS Secrets Manager, and using
that secret as an authentication token inside an Insomnia request.

> **Note:** Insomnia also supports connecting to AWS Secrets Manager directly through
> its built-in **External Vault** integration (no CLI steps required). See
> [Section 4 – Insomnia Native Vault Integration](#4-optional-insomnia-native-vault-integration)
> for that option. The CLI approach covered in Sections 1–3 works with any Insomnia
> tier and gives you full control using the AWS CLI.

---

## Prerequisites

| Requirement | Notes |
|---|---|
| AWS CLI v2 | Run `aws --version` to verify |
| AWS IAM Identity Center enabled | Must be enabled by an administrator |
| Kong Insomnia installed | Any current version |
| `jq` installed | Used to parse JSON from the CLI |

---

## Section 1 – Configure AWS CLI for SSO

To test it out I used SSO to connect to AWS.  If you choose one of the other methods, the steps will be similar.
I created a separate SSO session and profile named `insomnia` 

### Step 1: Find your SSO Start URL

Log in to the AWS Management Console and navigate to **IAM Identity Center**.
Your **AWS access portal URL** is displayed on the dashboard.

> Example: `https://d-xxxxxxxxxx.awsapps.com/start`

### Step 2: Create an SSO session and profile

Open a terminal and run:

```bash
aws configure set sso_session.insomnia-sso.sso_start_url https://<your-sso-start-url>/start
aws configure set sso_session.insomnia-sso.sso_region <your-sso-region>
aws configure set profile.insomnia.sso_session insomnia-sso
aws configure set profile.insomnia.sso_account_id <your-account-id>
aws configure set profile.insomnia.sso_role_name <your-role-name>
aws configure set profile.insomnia.region <your-default-region>
aws configure set profile.insomnia.output json
```

> **Important:** The SSO session name must **not contain spaces**. Use only letters,
> numbers, hyphens (`-`), and underscores (`_`).

Verify the result looks like this in `~/.aws/config`:

```ini
[sso-session insomnia-sso]
sso_start_url = https://<your-sso-start-url>/start
sso_region = <your-sso-region>

[profile insomnia]
sso_session = insomnia-sso
region = <your-default-region>
output = json
sso_account_id = <your-account-id>
sso_role_name = <your-role-name>
```

### Step 3: Log in with AWS SSO

```bash
aws sso login --profile insomnia
```

This opens your browser. Authenticate with your AWS credentials.

### Step 4: Validate the authenticated identity

```bash
aws sts get-caller-identity --profile insomnia
```

Expected output confirms your account ID and assumed role ARN.

---

## Section 2 – Create and Retrieve a Secret in AWS Secrets Manager

### Step 5: Create a secret

```bash
aws secretsmanager create-secret \
  --name my/secret/key \
  --secret-string '{"APIKEY":"<your-secret-value>"}' \
  --region <your-default-region> \
  --profile insomnia
```

To update an existing secret:

```bash
aws secretsmanager put-secret-value \
  --secret-id my/secret/key \
  --secret-string '{"APIKEY":"<your-secret-value>"}' \
  --region <your-default-region> \
  --profile insomnia
```

### Step 6: Retrieve and verify the secret

Retrieve the full secret:

```bash
aws secretsmanager get-secret-value \
  --secret-id my/secret/key \
  --region <your-default-region> \
  --profile insomnia \
  --query SecretString \
  --output text
```

---

## Section 3 – Connect Insomnia to AWS Using AWS Credentials

### Step 7: Export temporary credentials to your shell

The temporary credentials from your SSO session must be available in the environment
that runs Insomnia.

```bash
eval "$(aws configure export-credentials --profile insomnia --format env)"
```

Verify:

```bash
echo "$AWS_ACCESS_KEY_ID"
```

### Step 8: Configure Credentials in Insomnia

```bash
open -a Insomnia
```

From the Account dropdown found in the upper right corner of the insomnia app, select "Preferences". Then navigate click the "Credentials"

![Credentials Preferences](images/Insomnia1.png)

Under Service Provider Credential List, click "Add Credentials" and select "AWS".  Fill in the required fields for your chosen credential type and save or update.

![AWS Credentials](images/Insomnia2.png)

### Step 9 – Configure Insomnia: Setting the Environment Variable

In this step we will set up an environment variable in Insomnia to hold the secret value retrieved from AWS Secrets Manager. This allows us to reference the secret dynamically in our requests.

1) Go into a existing collection or create a new one. 

2) Then click on the "Base Environment" on the top left hand side of Insomnia.

![Base Environments](images/Insomnia3.png)

3) The Manage Environments window will open. Environment variable can be added individually by clicking "Add" button or Toggle the Table View to paste or edit using JSON Format.

![Edit Environments](images/Insomnia4.png)

4) Enter a Name for you variable.  You will use the name later to reference the secret.  Where you want the AWS Secret value stored, Type "aws". the aws vault function should appear then select Enter.  This will add the funcion to the field and it will appear in red.

![AWS Vault function1](images/Insomnia5.png)![AWS Vault Function2](images/Insomnia6.png)

5) Click the AWS Function and a menu will appear. Fill in the required fields" including the Crednetials Name created in Step 8, Service Name or ARN.  The Live Preview will show the value being pulled from AWS Secrets Manager. 

![AWS Function configuration](images/Insomnia7.png)

6) When finished,  Click "Done" to apply the changes. Then click "Close" to exit the Manage Environments window.

## Step 10 - Assigning your variable to a request

1) Create or select the request where you want the secret to be used.  

2) In the request, reference the environment variable.  In the example below, I am pulling the Secret into the API Key VALUE of the Authorization header.  To view the available environment variables available just type an _ and the list will appear.  Select the variable you created in Step 9. 

![Assigmomg Your Variable](images/Insomnia8.png)

## Step 11 – Execute the Request an view the results

Click the send button to execute the request.  If everything is configured correctly, the request will be sent with the secret value retrieved from AWS Secrets Manager and you should see a successful response in the console window.

![Run a test](images/Insomnia9.png)
---
