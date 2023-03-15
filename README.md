# 0. Introduction

This guide demonstrates how Conjur can be used to provide secrets such as AWS access keys and OS credentials to Terraform

Usage scenarios:

|No|Scenario|Description|
|---|---|---|
|1.|Hardcoded credentials|Secrets are passed to the providers via attributes on the manifest file or environment variables|
|2.|Using Conjur provider|Secrets are retrieved from Conjur via the Conjur provider and placed in data resources<br>Subsequent blocks can use the secrets by referencing the `value` attributes of the data resources|
|3.|Integrated with GitLab|Using Terraform with GitLab allows us to leverage on the [JWT integration between Conjur and GitLab](https://github.com/joetanx/conjur-gitlab)<br>Secrets can be retrieved from Conjur during an initial pipeline stage and passed on to Terraform on a subsequent pipeline stage via a pipeline artifact<br>Leveraging on the JWT integration also allows each Terraform project in GitLab to be uniquely identified and authenticated|

### Software Versions

|Software|Version|
|---|---|
|Terraform|1.4.0|
|Conjur Enterprise|12.9.0|
|RHEL|9.1|
|Podman|4.2|
|GitLab|15.9.3|

### Install Terraform

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

# 2. Use case 1: Create S3 bucket in AWS

## 2.1. Hardcoded credentials

### 2.1.1. AWS provider configuration

The AWS credentials are passed to the [AWS provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs) via either attributes on the manifest file and environment variables

#### 2.1.1.1. Environment variables

```console
export AWS_REGION="ap-southeast-1"
export AWS_ACCESS_KEY_ID="<aws_access_key_id>"
export AWS_SECRET_ACCESS_KEY="<aws_secret_access_key>"
```

#### 2.1.1.2. Attributes on the manifest file

```terraform
provider "aws" {
  region     = "ap-southeast-1"
  access_key = "<aws_access_key_id>"
  secret_key = "<aws_secret_access_key>"
}
```

### 2.1.2. Example Terraform manifest:

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

### 2.1.3. Testing the example manifest

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

## 2.2. Using Conjur provider

### 2.2.1. Conjur provider configuration

The parameters required for Conjur communications are passed to the [Conjur provider](https://registry.terraform.io/providers/cyberark/conjur/latest/docs) via either attributes on the manifest file and environment variables

#### 2.2.1.1. Environment variables

```console
export CONJUR_APPLIANCE_URL="https://<conjur-service-fqdn>"
export CONJUR_ACCOUNT="<conjur-account-name>"
export CONJUR_AUTHN_LOGIN="<conjur-host-identity>"
export CONJUR_AUTHN_API_KEY="<conjur-host-api-key>"
export CONJUR_CERT_FILE="<path-to-conjur-or-CA-certificate>"
```

#### 2.2.1.2. Attributes on the manifest file

```terraform
provider "conjur" {
  appliance_url = "https://<conjur-service-fqdn>"
  account = "<conjur-account-name>"
  login="<conjur-host-identity>"
  api_key="<conjur-host-api-key>"
  ssl_cert="<path-to-conjur-or-CA-certificate>"
}
```

### 2.2.2. Example Terraform manifest:

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

### 2.2.3. Testing the example manifest

Same steps as [2.1.3.](#213-testing-the-example-manifest)

## 2.3. Integrated with GitLab

# . Deploy IIS web server using `remote-exec` (`winrm`)

## .1. Hardcoded credentials

## .2. Using Conjur provider

## .3. Integrated with GitLab

# . Deploy apache web server using `remote-exec` (`ssh`)

## .1. Hardcoded credentials

## .2. Using Conjur provider

## .3. Integrated with GitLab
