# Hail Packer Builds

This ReadMe contains expanded documentation surrounding the AMI custom build process referenced in the [Deployment Guide](/readme.md#deployment-guide).

Hail on EMR requires the use of a custom AMI with Hail, Spark, VEP, and reference genomes preconfigured.  This build process is driven by Packer, and leverages AWS CodeBuild.  Note that some of these software packages are optional, and the build process can be executed for different versions or combinations of these software packages.

[Public AMIs](/readme.md#public-amis) are published and referenced in the root of this repo.  These AMIs are built using this workflow.  If you wish to create you own custom AMIs, follow the process documented here.

_Note:  Creating these custom AMIs is a complicated process and requires working knowledge of AWS CodeBuild, Packer from Hashicorp, and shell scripting.  Troubleshooting will require intimiate knowledge of Hail, VEP, and their associated build processes (including knowledge of perl).  Proceed with caution._

## Table of Contents

- [Hail Packer Builds](#hail-packer-builds)
  - [Directory Structure](#directory-structure)
    - [Builds](#builds)
    - [CodeBuild](#codebuild)
    - [Scripts](#scripts)
  - [CLI Building](#cli-building)
    - [Prerequisites](#prerequisites)
    - [Execution](#execution)
  - [Troubleshooting](#troubleshooting)
    - [AMI Exists](#ami-exists)


## Directory Structure

Below is a diagram of the directory structure, followed by a description of the contents of each directory.

```tree -L 3
.
├── amazon-linux.json
├── builds
│   ├── emr-5.25.0.vars
│   └── vpcs
│       └── vpc01.vars.example
├── build-wrapper.sh
├── codebuild
│   └── buildspec.yml
├── docs
│   ├── hail-ami.md
│   └── images
│       ├── codebuild_running.png
│       └── codebuild_start.png
├── ReadMe.md
└── scripts
    ├── ami_cleanup.sh
    ├── hail_build.sh
    ├── htslib.sh
    ├── R_install.R
    ├── samtools.sh
    └── vep_install.sh
```

### Builds

The `builds` directory contains variable definitions that are used as arguments to the primary build JSON, [amazon-linux.json](amazon-linux.json).

Target VPCs, which may exist in remote accounts, are broken out into individual files under `builds/vpcs`.  There is an example VPC var file in the repository.  Note that after any adjustments to the settings or files in `builds/vpcs` you will need to update the path to that file in the [hail-ami.yml](../hail-ami.yml) CloudFormation template for the appropriate CodeBuild project.

### CodeBuild

The `codebuild` directory contains configuration items related to AWS CodeBuild, which executes the Packer build process to build the custom AMIs.  The [buildspec.yml](codebuild/buildspec.yml) file contains instructions for Packer installation and invocation via AWS CodeBuild.  See [the Hail AMI markdown](docs/hail-ami.md) for more details.

### Scripts

The `scripts` directory contains bash scripts supporting the build components (VEP, Hail, supporting python packages, etc.).  These scripts may be referenced from [amazon-linux.json](amazon-linux.json).  Scripts in this directory are linted with [ShellCheck](https://github.com/koalaman/shellcheck).


## CLI Building
To execute a build using the CLI, follow these steps.

### Prerequisites

- AWS IAM profile configured in your shell with a **region** defined. This profile **must have permission to launch instances and create AMIs**.  The majority of required IAM permissions are documented [here](https://www.packer.io/docs/builders/amazon.html).  [iam:PassRole](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_passrole.html) is also required for the CodeBuild service role to associate the hail-packer EC2 instance role to the launched instance.
- Shell `AWS_PROFILE` exported
- [Packer installed](https://www.packer.io/downloads.html) -  Version 1.4 or greater is recommended.

### Execution

Builds are executed via the [build wrapper](build-wrapper.sh).  This wrapper has two methods of invocation:

- AWS CodeBuild

  Environment variables are defined in the CodeBuild Project and the script is invoked with no arguments.  See [the Hail AMI markdown](docs/hail-ami.md) for more details.

- Standard CLI

  Arguments are added via the CLI. Use `./build-wrapper --help` for more details.

  ```bash
  usage: build-wrapper.sh [ARGUMENTS]

    --hail-version  [Number Version]    - OPTIONAL.  If omitted, the current HEAD of master branch will be pulled.
    --vep-version   [Number Version]    - OPTIONAL.  If omitted, VEP will not be included.
    --hail-bucket   [S3 Bucket Name]    - REQUIRED
    --var-file      [Full File Path]    - REQUIRED
    --vpc-var-file  [Full File Path]    - REQUIRED

    Example:

   build-wrapper.sh --hail-version 0.2.18 \
                    --vep-version 96 \
                    --hail-bucket YOUR_HAIL_BUCKET \
                    --var-file builds/emr-5.25.0.vars \
                    --vpc-var-file builds/vpcs/account123-vpc01.vars
  ```

## Troubleshooting

### AMI Exists

AMI names are unique.  In order to rebuild an AMI with the same name you will need to deregister the AMI your AWS account and target region.
