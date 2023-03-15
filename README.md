# 0. Introduction

This guide demonstrates how Conjur can be used to provide secrets such as AWS access keys and OS credentials to Terraform

### 0.1. Scenarios demonstrated:

|No|Scenario|Description|
|---|---|---|
|1.|Hardcoded credentials|Secrets are passed to the providers via attributes on the manifest file or environment variables|
|2.|Using Conjur provider|• Secrets are retrieved from Conjur via the Conjur provider and placed in data resources<br>• Subsequent blocks can use the secrets by referencing the `value` attributes of the data resources|
|3.|Integrated with GitLab|• Using Terraform with GitLab allows us to leverage on the [JWT integration between Conjur and GitLab](https://github.com/joetanx/conjur-gitlab)<br>• Secrets can be retrieved from Conjur during an initial pipeline stage and passed on to Terraform on a subsequent pipeline stage via a pipeline artifact<br>• Leveraging on the JWT integration also allows each Terraform project in GitLab to be uniquely identified and authenticated|

### 0.2. Software Versions

|Software|Version|
|---|---|
|Terraform|1.4.0|
|Conjur Enterprise|12.9.0|
|RHEL|9.1|
|Podman|4.2|
|GitLab|15.9.3|

### 0.3. Install Terraform

Teraform can be installed on RHEL in just a few commands:

```console
curl -L -o /etc/yum.repos.d/hashicorp.repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
yum -y install terraform
rm -f /etc/yum.repos.d/hashicorp.repo
```

# 1. Integration with Conjur

This section assumes that the Conjur environment is already available.

Otherwise, setup Conjur master according to this guide: https://github.com/joetanx/setup/blob/main/conjur.md

## 1.1. Setup Conjur policy

- Load the Conjur policy `tf-vars.yaml`
  - Creates the policy `PLACEHOLDER`
    - Creates variables `PLACEHOLDER` and `PLACEHOLDER` to contain credentials for the Ansible managed node
    - Creates `consumers` group to authorize members of this group to access the variables
  - Creates the policy `PLACEHOLDER` with a same-name layer and a host `demo`
    - The AAP server will use the Conjur identity `host/ansible/demo` to retrieve credentials
    - Adds `ansible` layer to `consumers` group for `ssh_keys` policy

```console
curl -O https://raw.githubusercontent.com/joetanx/conjur-terraform/main/tf-vars.yaml
conjur policy load -b root -f tf-vars.yaml && rm -f tf-vars.yaml
```

- **Note** ☝️ : the API key of the Conjur identity `host/ansible/demo` will be shown on console after loading the policy, this key is required to configure Conjur as external secrets management system in [4.3.](#43-configure-conjur-as-an-external-secrets-management-system)

# 2. Hardcoded credentials

## 2.1. AWS provider configuration

The AWS credentials are passed to the [AWS provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs) via either attributes on the manifest file and environment variables

### 2.1.1. Environment variables

```console
export AWS_REGION="ap-southeast-1"
export AWS_ACCESS_KEY_ID="<aws_access_key_id>"
export AWS_SECRET_ACCESS_KEY="<aws_secret_access_key>"
```

### 2.1.2. Attributes on the manifest file

```terraform
provider "aws" {
  region     = "ap-southeast-1"
  access_key = "<aws_access_key_id>"
  secret_key = "<aws_secret_access_key>"
}
```

## 2.2. Example Terraform manifest

- Creates a S3 bucket with public access disabled
- Uploads [demo.txt](/demo.txt) to the newly created bucket

```terraform
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}

provider "aws" {
  # <edit here as necessary>
}

resource "aws_s3_bucket" "demo" {
  bucket = "<my-bucket-name>"
}

resource "aws_s3_bucket_public_access_block" "demo" {
  bucket = aws_s3_bucket.demo.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_object" "demo" {
  bucket = aws_s3_bucket.demo.id
  key    = "demo.txt"
  source = "demo.txt"
  etag = filemd5("demo.txt")
}

output "id" {
    value = aws_s3_bucket.demo.id
}

output "arn" {
    value = aws_s3_bucket.demo.arn
}
```

## 2.3. Testing the example manifest

1. Create Terraform working directory and `cd` into it: `mkdir tfdemo && cd $_`
2. Edit the manifest with your AWS credentials and the bucket name and save it as `main.tf` in the working directory
3. Download [demo.txt](/demo.txt) into the working directory: `curl -O https://raw.githubusercontent.com/joetanx/conjur-terraform/main/demo.txt`
4. Initialize Terraform working directory: `terraform init`
5. Apply the manifest: `terraform apply -auto-approve`
6. Verify the created bucket

    Example output:

    ```console
    [root@gitlab ~]# aws s3 ls s3://jtan-tfdemo
    2023-03-15 09:39:30        744 demo.txt
    [root@gitlab ~]# aws s3 cp s3://jtan-tfdemo/demo.txt .
    download: s3://jtan-tfdemo/demo.txt to ./demo.txt
    [root@gitlab ~]# cat ./demo.txt
       ____      _                _         _      ____                     _         __  __
      / ___|   _| |__   ___ _ __ / \   _ __| | __ / ___|  ___  ___ _ __ ___| |_ ___  |  \/  | __ _ _ __   __ _  __ _  ___ _ __
     | |  | | | | '_ \ / _ \ '__/ _ \ | '__| |/ / \___ \ / _ \/ __| '__/ _ \ __/ __| | |\/| |/ _` | '_ \ / _` |/ _` |/ _ \ '__|
     | |__| |_| | |_) |  __/ | / ___ \| |  |   <   ___) |  __/ (__| | |  __/ |_\__ \ | |  | | (_| | | | | (_| | (_| |  __/ |
      \____\__, |_.__/ \___|_|/_/   \_\_|  |_|\_\ |____/ \___|\___|_|  \___|\__|___/ |_|  |_|\__,_|_| |_|\__,_|\__, |\___|_|
           |___/                                                                                               |___/
    ```

7. Clean-up the deployed components: `terraform destroy -auto-approve`

# 3. Using Conjur provider

## 3.1. Conjur provider configuration

The parameters required for Conjur communications are passed to the [Conjur provider](https://registry.terraform.io/providers/cyberark/conjur/latest/docs) via either attributes on the manifest file and environment variables

### 3.1.1. Environment variables

```console
export CONJUR_APPLIANCE_URL="https://<conjur-service-fqdn>"
export CONJUR_ACCOUNT="<conjur-account-name>"
export CONJUR_AUTHN_LOGIN="<conjur-host-identity>"
export CONJUR_AUTHN_API_KEY="<conjur-host-api-key>"
export CONJUR_CERT_FILE="<path-to-conjur-or-CA-certificate>"
```

### 3.1.2. Attributes on the manifest file

```terraform
provider "conjur" {
  appliance_url = "https://<conjur-service-fqdn>"
  account = "<conjur-account-name>"
  login="<conjur-host-identity>"
  api_key="<conjur-host-api-key>"
  ssl_cert="<path-to-conjur-or-CA-certificate>"
}
```

## 3.2. Example Terraform manifest

```terraform
terraform {
  required_providers {
    conjur = {
      source = "cyberark/conjur"
      version = "~> 0.0"
    }
    aws = {
      source = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}

provider "conjur" {
  # <edit here as necessary>
}

data "conjur_secret" "awsakid" {
  name = "aws_api/awsakid"
}

data "conjur_secret" "awssak" {
  name = "aws_api/awssak"
}

provider "aws" {
  region     = "ap-southeast-1"
  access_key = "${data.conjur_secret.awsakid.value}"
  secret_key = "${data.conjur_secret.awssak.value}"
}

resource "aws_s3_bucket" "demo" {
  bucket = "<my-bucket-name>"
}

resource "aws_s3_bucket_public_access_block" "demo" {
  bucket = aws_s3_bucket.demo.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_object" "demo" {
  bucket = aws_s3_bucket.demo.id
  key    = "demo.txt"
  source = "demo.txt"
  etag = filemd5("demo.txt")
}

output "id" {
    value = aws_s3_bucket.demo.id
}

output "arn" {
    value = aws_s3_bucket.demo.arn
}
```

## 3.3. Testing the example manifest

Same steps as [2.3.](#23-testing-the-example-manifest)

# 4. Integrated with GitLab

## 4.1. GitLab Integration

GitLab integrates with Conjur via the JWT authenticator, which establishes a trust between Conjur and GitLab via the JSON Web Key Set (JWKS)
- Each project on GitLab retrieving credentials will have its JWT signed and verified via the JWKS
- This mitigates the "secret-zero" problem and enable each project on GitLab to be uniquely identified

Terraform has in-depth [integration with GitLab](https://docs.gitlab.com/ee/user/infrastructure/iac/) that allows
- Terraform to run in GitLab docker executor runner
- Using GitLab as Terraform state backend

Leveraging on the transitive integration relationships of Conjur ↔ GitLab ↔ Terraform, a more secure method of secrets management can be achieved

### 4.2. Preparing GitLab

The GitLab-Conjur Integration is documented here: https://github.com/joetanx/conjur-gitlab, relevant sections are linked below for convenient access:

- [How does GitLab integration with Conjur using JWT work?](https://github.com/joetanx/conjur-gitlab#how-does-gitlab-integration-with-conjur-using-jwt-work)
- [Setup GitLab](https://github.com/joetanx/conjur-gitlab#3-setup-gitlab)
- [Conjur policies for GitLab JWT](https://github.com/joetanx/conjur-gitlab#4-conjur-policies-for-gitlab-jwt)

## 4.3. Configure Terraform project in GitLab

### 4.3.1. Example Terraform manifest

☝️ Notice that the AWS provider configuration is empty (`provider "aws" {}`), since it takes the environment variables from the previous stage in GitLab

```terraform
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}

provider "aws" {}

resource "aws_s3_bucket" "demo" {
  bucket = "jtan-tfdemo"
}

resource "aws_s3_bucket_public_access_block" "demo" {
  bucket = aws_s3_bucket.demo.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_object" "demo" {
  bucket = aws_s3_bucket.demo.id
  key    = "demo.txt"
  source = "demo.txt"
  etag = filemd5("demo.txt")
}

output "id" {
    value = aws_s3_bucket.demo.id
}

output "arn" {
    value = aws_s3_bucket.demo.arn
}

### 4.3.2. Example GitLab pipeline

```yaml
variables:
  AWS_REGION: ap-southeast-1
  CONJUR_APPLIANCE_URL: "https://conjur.vx"
  CONJUR_ACCOUNT: "cyberark"
  CONJUR_AUTHN_JWT_SERVICE_ID: "gitlab"
  CONJUR_AUTHN_JWT_TOKEN: "${CI_JOB_JWT_V2}"
  CONJUR_CERT_FILE: "central.pem"
Fetch variables from Conjur:
  image:
    name: docker.io/nfmsjoeg/authn-jwt-gitlab:latest
  stage: .pre
  script:
    - echo AWS_ACCESS_KEY_ID=$(CONJUR_SECRET_ID="aws_api/awsakid" /authn-jwt-gitlab) >> conjurVariables.env
    - echo AWS_SECRET_ACCESS_KEY=$(CONJUR_SECRET_ID="aws_api/awssak" /authn-jwt-gitlab) >> conjurVariables.env
  artifacts:
    reports:
      dotenv: conjurVariables.env
Check caller AWS STS token via Terraform using variables from Conjur:
  image:
    name: docker.io/hashicorp/terraform:latest
    entrypoint: [""]
  before_script:
    - terraform init
  script:
    - terraform plan
```

### 4.4. Clean-up job

```
# https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket
# Check if bucket name exist: curl -I https://bucket.s3.amazonaws.com | grep bucket-region
# List content in bucket: aws s3 ls s3://jtan-tfdemo
# Delete bucket with AWS CLI: aws s3 rb s3://jtan-tfdemo --force
```

# 5. Other Terraform examples

## 5.1. Deploy IIS web server using `remote-exec` (`winrm`)

## 5.2. Deploy apache web server using `remote-exec` (`ssh`)
