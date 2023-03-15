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

## 0.4. Integration with Conjur

This guide assumes that the Conjur environment is already available.

Otherwise, setup a single-node Conjur according to this guide: https://github.com/joetanx/setup/blob/main/conjur.md

# 1. Example: create S3 bucket

|File|Purpose|
|---|---|
|`main.tf`|• Creates S3 bucket<br>• Uploads test file `demo.txt` to bucket|
|`provider.tf`|Configures information for the providers to work|

## 1.1. main.tf

```terraform
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
```

# 2. Hardcoded credentials

The AWS credentials are passed to the [AWS provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs) via either environment variables or attributes on the manifest file

## 2.1. Using bash environment varibles

Export AWS credentials into bash environment:

```console
export AWS_REGION="ap-southeast-1"
export AWS_ACCESS_KEY_ID="<aws_access_key_id>"
export AWS_SECRET_ACCESS_KEY="<aws_secret_access_key>"
```

Content for `provider.tf`:

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
```

## 2.2. Using attributes on manifest file

Content for `provider.tf`:

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
  region     = "ap-southeast-1"
  access_key = "<aws_access_key_id>"
  secret_key = "<aws_secret_access_key>"
}
```

## 2.3. Testing the example manifest

1. Create Terraform working directory and `cd` into it: `mkdir tfdemo && cd $_`
2. Create and edit the `main.tf` and `provider.tf` file in the working directory
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

# 3 Using Conjur provider

## 3.0. Preparing for Conjur integration

### 3.0.1. Setup Conjur policy

The Conjur policy `tf-vars.yaml` builds on top of the [app-vars.yaml](https://github.com/joetanx/setup/blob/main/app-vars.yaml) which is loaded in the [Conjur setup guide](https://github.com/joetanx/setup/blob/main/conjur.md)

This `tf-vars.yaml` policy does the following:

- Creates the policy `linux` and `windows` for the `remote-exec` example in [section 5](#5-example-remote-exec-provisioner)
- Creates the policy `terraform` and host `demo` under it for use with the Terraform Conjur provider
  - Grants host `terraform/demo` access to variables in `aws_api`, `linux` and `windows` policies by adding it to `consumers` group

```console
curl -O https://raw.githubusercontent.com/joetanx/conjur-terraform/main/tf-vars.yaml
conjur policy load -b root -f tf-vars.yaml && rm -f tf-vars.yaml
```

☝️ The API key of the Conjur identity `host/terraform/demo` will be shown on console after loading the policy, this key is required for the Terraform Conjur provider

### 3.0.2. Configuring the Terraform Conjur provider

The parameters required for Conjur communications are passed to the [Conjur provider](https://registry.terraform.io/providers/cyberark/conjur/latest/docs) via either attributes on the manifest file and environment variables

|Parameter|Description|
|---|---|
|Conjur URL|The URL of the Conjur cluster or followers|
|Conjur account|The account name of the Conjur cluster|
|Conjur authentication login|The host identity configured in Conjur policy|
|Conjur authentication API key|The API generated when the host identity is created|
|Conjur certificate path|The Conjur certificate or the CA certificate which signed the Conjur certificate is required, edit the certificate path variable or attribute to point to this file|

## 3.1. Using bash environment varibles

Export Conjur parameters into bash environment:

```console
export CONJUR_APPLIANCE_URL="https://<conjur-service-fqdn>"
export CONJUR_ACCOUNT="<conjur-account-name>"
export CONJUR_AUTHN_LOGIN="<conjur-host-identity>"
export CONJUR_AUTHN_API_KEY="<conjur-host-api-key>"
export CONJUR_CERT_FILE="<path-to-conjur-or-CA-certificate>"
```

Content for `provider.tf`:

```terraform
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      version = "~> 4.0"
    }
    conjur = {
      source = "cyberark/conjur"
      version = "~> 0.0"
    }
  }
}

provider "conjur" {}

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
```

## 3.2. Using attributes on manifest file

Content for `provider.tf`:

```terraform
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      version = "~> 4.0"
    }
    conjur = {
      source = "cyberark/conjur"
      version = "~> 0.0"
    }
  }
}

provider "conjur" {
  appliance_url = "https://<conjur-service-fqdn>"
  account       = "<conjur-account-name>"
  login         = "<conjur-host-identity>"
  api_key       = "<conjur-host-api-key>"
  ssl_cert_path = "<path-to-conjur-or-CA-certificate>"
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
```

## 3.3. Testing the example manifest

Same steps as [2.3.](#23-testing-the-example-manifest)

# 4. Integrated with GitLab

GitLab integrates with Conjur via the JWT authenticator, which establishes a trust between Conjur and GitLab via the JSON Web Key Set (JWKS)
- Each project on GitLab retrieving credentials will have its JWT signed and verified via the JWKS
- This mitigates the "secret-zero" problem and enable each project on GitLab to be uniquely identified

Terraform has in-depth [integration with GitLab](https://docs.gitlab.com/ee/user/infrastructure/iac/) that allows
- Terraform to run in GitLab docker executor runner
- Using GitLab as Terraform state backend

Leveraging on the transitive integration relationships of Conjur ↔ GitLab ↔ Terraform, a more secure method of secrets management can be achieved

## 4.1. Preparing GitLab

The GitLab-Conjur Integration is documented here: https://github.com/joetanx/conjur-gitlab, relevant sections are linked below for convenient access:

- [How does GitLab integration with Conjur using JWT work?](https://github.com/joetanx/conjur-gitlab#how-does-gitlab-integration-with-conjur-using-jwt-work)
- [Setup GitLab](https://github.com/joetanx/conjur-gitlab#3-setup-gitlab)

### 4.1.1. Setup Conjur policy

The Conjur policy `tf-gitlab-vars.yaml` builds on top of `authn-jwt-hosts.yaml` and `authn-jwt.yaml` in the [GitLab integration guide](https://github.com/joetanx/conjur-gitlab)

This `tf-gitlab-vars.yaml` policy does the following:

- Add GitLab projects as hosts into the `jwt-apps/gitlab` policy
  - The `jwt-apps/gitlab` policy is created in https://github.com/joetanx/conjur-gitlab
  - The hosts are added to the same-name layer, which is added to the JWT authenticator's `consumers` group to allow them to authenticate to the JWT web service
  - Respective hosts are granted access to variables in `aws_api`, `linux` and `windows` policies by adding them to `consumers` group

|GitLab project name|Conjur host identity|
|---|---|
|Terraform AWS S3 Demo|`cybr/terraform-aws-s3-demo`|
|Terraform AWS S3 Cleanup|`cybr/terraform-aws-s3-cleanup`|
|Terraform remote-exec SSH Demo|`cybr/terraform-remote-exec-ssh-demo`|
|Terraform remote-exec WinRM Demo|`cybr/terraform-remote-exec-winrm-demo`|

☝️ Project names are important! Remember that for GitLab JWT authentication to work, the `project path` must match the `host identity` configured in the Conjur policy

```console
curl -O https://raw.githubusercontent.com/joetanx/conjur-terraform/main/tf-gitlab-vars.yaml
conjur policy load -b root -f tf-gitlab-vars.yaml && rm -f tf-gitlab-vars.yaml
```

## 4.2. Configure Terraform project in GitLab

GitLab project name: `Terraform AWS S3 Demo`

☝️ Project name is important! Remember that for GitLab JWT authentication to work, the `project path` must match the `host identity` configured in the Conjur policy

### 4.2.1. main.tf and demo.txt

- `main.tf`: from step [1.1. main.tf](#11-maintf)
- `demo.txt`: from [demo.txt](/demo.txt) file in this repository (or really just any text will do)

### 4.2.3. provider.tf

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
```

### 4.2.4. .gitlab-ci.yml

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
  stage: deploy
  before_script:
    - terraform init
  script:
    - terraform plan
```

## 4.3. Use another GitLab project to verify and delete the S3 bucket (because we can =D)

GitLab project name: `Terraform AWS S3 Cleanup`

☝️ Project name is important! Remember that for GitLab JWT authentication to work, the `project path` must match the `host identity` configured in the Conjur policy

### 4.3.1. .gitlab-ci.yml

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
Verify bucket:
  stage: test
  image:
    name: docker.io/amazon/aws-cli:latest
    entrypoint: [""]
  script:
    - aws s3 ls s3://jtan-tfdemo
    - aws s3 cp s3://jtan-tfdemo/demo.txt .
    - cat ./demo.txt
Delete bucket:
  stage: deploy
  image:
    name: docker.io/amazon/aws-cli:latest
    entrypoint: [""]
  script:
    - aws s3 rb s3://jtan-tfdemo --force
  rules:
    - when: manual
```

# 5. Example: remote-exec provisioner

Terraform is also able to configure deployed cloud instances using provisioners

This section is a code dump of my example configuration for using `remote-exec` to deploy Apache (Linux) or IIS (Windows) web servers

## 5.1. Linux (remote-exec/ssh)

GitLab project name: `Terraform remote-exec SSH Demo`

☝️ Project name is important! Remember that for GitLab JWT authentication to work, the `project path` must match the `host identity` configured in the Conjur policy

### 5.1.1. index.html

From [index.html](/index.html) file in this repository (or really just any text will do)

### 5.1.2. main.tf

```terraform
variable "LINUX_USERNAME" {
  type = string
}

variable "LINUX_PASSWORD" {
  type = string
}

resource "null_resource" "SSHTest" {
  connection {
    type     = "ssh"
    user     = var.LINUX_USERNAME
    password = var.LINUX_PASSWORD
    host     = "foxtrot.vx"
    port     = 22
    timeout  = "20s"
  }
  provisioner "remote-exec" {
    inline = [
      "yum -y install httpd",
      "systemctl enable --now httpd",
      "firewall-cmd --add-service http --permanent",
      "firewall-cmd --reload"
    ]
  }
  provisioner "file" {
    source      = "index.html"
    destination = "/var/www/html/index.html"
  }
}
```

### 5.1.3. provider.tf

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
```

### 5.1.4. .gitlab-ci.yml

```yaml
variables:
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
    - echo TF_VAR_LINUX_USERNAME=$(CONJUR_SECRET_ID="linux/username" /authn-jwt-gitlab) >> conjurVariables.env
    - echo TF_VAR_LINUX_PASSWORD=$(CONJUR_SECRET_ID="linux/password" /authn-jwt-gitlab) >> conjurVariables.env
  artifacts:
    reports:
      dotenv: conjurVariables.env
Run Terraform using variables from Conjur:
  image:
    name: docker.io/hashicorp/terraform:latest
    entrypoint: [""]
  stage: deploy
  before_script:
    - terraform init
  script:
    - terraform apply -auto-approve
```

## 5.2. Windows (remote-exec/winrm)

GitLab project name: `Terraform remote-exec WinRM Demo`

☝️ Project name is important! Remember that for GitLab JWT authentication to work, the `project path` must match the `host identity` configured in the Conjur policy

### 5.2.1. index.html

From [index.html](/index.html) file in this repository (or really just any text will do)

### 5.2.2. main.tf

```terraform
variable "WINDOWS_USERNAME" {
  type = string
}

variable "WINDOWS_PASSWORD" {
  type = string
}

resource "null_resource" "WinRMTest" {
  connection {
    type     = "winrm"
    user     = var.WINDOWS_USERNAME
    password = var.WINDOWS_PASSWORD
    host     = "quebec.vx"
    port     = 5985
    timeout  = "2m"
    use_ntlm = true
  }
  provisioner "remote-exec" {
    inline = [
      "powershell -NoProfile -Command Install-WindowsFeature -name Web-Server -IncludeManagementTools"
    ]
  }
  provisioner "file" {
    source      = "index.html"
    destination = "C:\\inetpub\\wwwroot\\index.html"
  }
}
```

### 5.2.3. provider.tf

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
```

### 5.2.4. .gitlab-ci.yml

```yaml
variables:
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
    - echo TF_VAR_WINDOWS_USERNAME=$(CONJUR_SECRET_ID="windows/username" /authn-jwt-gitlab) >> conjurVariables.env
    - echo TF_VAR_WINDOWS_PASSWORD=$(CONJUR_SECRET_ID="windows/password" /authn-jwt-gitlab) >> conjurVariables.env
  artifacts:
    reports:
      dotenv: conjurVariables.env
Run Terraform using variables from Conjur:
  image:
    name: docker.io/hashicorp/terraform:latest
    entrypoint: [""]
  stage: deploy
  before_script:
    - terraform init
  script:
    - terraform apply -auto-approve
```
