# Class 7.5 / SEIR-1: Week 4 Prep Notes

---

## Table of Contents

- [Introduction](#introduction)
- [Why Is This Tool Needed?](#why-is-this-tool-needed)
- [API](#api)
- [Google Cloud SDK](#google-cloud-sdk)
- [The Terraform Solution](#the-terraform-solution)
- [Terraform Workflow](#terraform-workflow)
- [Development Environment Setup](#development-environment-setup)
- [Exploring VS Code](#exploring-vs-code)
- [Basic Terraform](#basic-terraform)
- [The Google Provider and a Basic Infrastructure Example](#the-google-provider-and-a-basic-infrastructure-example)
- [Learning Resources](#learning-resources)

---

## Introduction

This week will be covering the fundamentals of Terraform. This is an automation tool for provisioning and managing cloud infrastructure and other tools. It is a tool in the category called IaC (Infrastructure as Code). It is the most popular IaC tool (by far) and is cloud agnostic, meaning it works in essentially any public cloud and even private clouds.

We will start from the ground up by showing why we need this tool, the issues it fixes, how to set it up, and eventually how to redo the Week 1/2 labs with Terraform from scratch.

### Class Overview

- Introduction
- Discuss prelims
- Discuss goals
- VS Code, HashiCorp docs
- Discuss the end goal
- Discuss Terraform features
- Discuss HCL syntax
- Build out code
- Terraform workflow
- Troubleshooting

### Preliminary Materials

The script checks the following:

- Google Cloud SDK (in particular, `gcloud`) is installed and authenticated
- Terraform binary is installed and up to date
- TheoWAF folder present at `~/Documents/TheoWAF/class7.5/GCP/Terraform`
- Creates a `.gitignore` file
- Verifies a JSON token is present in the `Terraform` directory

It will create the TheoWAF folder structure if needed and will download a `.gitignore` file configured for Terraform projects.

```bash
curl https://raw.githubusercontent.com/aaron-dm-mcdonald/class7.5-notes/refs/heads/main/week-4/prep/scripts/prelim.sh | sh
```

---

## Why Is This Tool Needed?

In enterprise environments, automation becomes critical and is one of the main tasks of DevOps. Using the cloud the way we have been so far is called ClickOps, since we click through the console to do things. Cloud infrastructure allows for a great deal of automation, so these two areas have a lot of crossover. Naturally, automating the creation of infrastructure in the cloud follows. There are also other reasons to use this tool, which become clear from the cons of ClickOps.

**Cons of ClickOps:**

- Difficult to reproduce across environments (dev, staging, prod)
- Not self-documenting: no record of what you clicked or why
- Impossible to automate or version control
- No single source of truth that is auditable

But *why* this tool specifically? First we should discuss more traditional means of automation, using the Week 1 lab as an example (leaving off the startup script for brevity).

---

## API

An API (Application Programming Interface) is at the heart of GCP and all cloud platforms. It allows programs, services, and computers to communicate with GCP. Every button in the GCP console actually makes API calls behind the scenes.

**API call example:**

Here we look at an HTTP request that could be sent to the GCP API. Realistically this would never be done manually, but it helps illustrate what is happening under the hood.

- A `POST` request is the opposite of loading a webpage; instead of getting data, you are sending it
- The API endpoint is a specific URL meant to receive a request and act on it; here we use the Compute Engine endpoint
- `Host` is the actual location of the API on the internet
- `HTTP/1.1` is the version of the HTTP protocol being used
- `Content-Type` declares that the payload is formatted as JSON
- `Content-Length` is the size of the request body in bytes
- `Authorization` is where you prove you are allowed to perform the action in GCP

```http
POST /compute/v1/projects/<YOUR_PROJECT_ID>/zones/<YOUR_ZONE>/instances HTTP/1.1
Host: compute.googleapis.com
Authorization: Bearer <YOUR_ACCESS_TOKEN>
Content-Type: application/json
Content-Length: 347

{
  "name": "<INSTANCE_NAME>",
  "machineType": "zones/<YOUR_ZONE>/machineTypes/e2-medium",
  "disks": [
    {
      "boot": true,
      "initializeParams": {
        "sourceImage": "projects/debian-cloud/global/images/family/debian-11"
      }
    }
  ],
  "networkInterfaces": [
    {
      "network": "global/networks/default"
    }
  ]
}
```

**Cons of direct API calls:**

- Extremely verbose and error-prone
- Requires manual authentication handling
- Not designed for human interaction
- No state tracking: you must manually track what you have created
- Hard to make small edits

**Reference:** https://docs.cloud.google.com/compute/docs/reference/rest/v1/instances/insert

---

## Google Cloud SDK

The Google Cloud SDK is a collection of tools and libraries for working with GCP. The most familiar tool in it is `gcloud`. While `gcloud` is the largest component, there are also many specialized tools and language-specific libraries. We will focus on `gcloud` for now.

`gcloud` is a CLI utility written in Python. It handles authentication when you run `gcloud init` -- your browser is opened and credentials are saved locally via SSO. It can do almost everything the console can do, and in some cases more. It also lets you preset a default project, region, and zone. Instead of the verbose API call above, the same VM creation looks like this:

```bash
gcloud compute instances create <INSTANCE_NAME> \
  --project=<YOUR_PROJECT_ID> \
  --zone=<YOUR_ZONE> \
  --machine-type=e2-medium \
  --image-family=debian-11 \
  --image-project=debian-cloud \
  --network=default
```

**Cons of the GCP CLI:**

- Many flags needed for many parameters
- Different commands needed for create, update, and delete operations
- Easy to make mistakes
- Very hard to reproduce
- No state: the CLI does not remember what resources it created
- No idempotency: running the same command twice creates duplicate resources
- Entirely imperative: you say what to do and in what order

**Pros of the GCP CLI:**

> Note: The CLI is not a bad tool- it just is not the ideal tool for automating infrastructure creation.

- Great for one-time tasks and quick admin work (wrapping SSH sessions, restarting a fleet of VMs)
- Easy to embed in shell scripts
- Very effective for querying and filtering resource information

---

## The Terraform Solution

Terraform solves the problems from the previous sections by:

- Using **declarative code**- describe what you want, not how to create it
- Being **stateful**- it knows what exists and what needs to change
- Being **idempotent**- safe to run multiple times if no configuration has changed
- **Managing dependencies automatically**- it figures out what needs to be created first, which matters a lot when you have 50 resources with dependencies between them
- Supporting **version control**
- Enabling **code reuse**

### What Is Terraform?

Terraform is an IaC tool. Infrastructure as Code means writing infrastructure definitions as actual code. Instead of clicking on things in the console you are testing code as you write along the way, following the software development lifecycle, but instead of application code, you are writing code that defines infrastructure.

Terraform lets you define, provision, query, and manage cloud resources using a declarative configuration language. Instead of clicking through the GCP console or writing complex CLI commands, you write code that describes your desired infrastructure, and Terraform handles all the API calls and sequencing needed to make it happen.

**Official documentation:** https://developer.hashicorp.com/terraform/docs

### Key Terms

- **IaC (Infrastructure as Code):** The practice of managing infrastructure through code rather than manual configuration. Terraform is an IaC tool that writes code to interact with cloud provider APIs.

- **Terraform:** An open-source IaC tool created by HashiCorp. Cloud agnostic (works with AWS, Azure, GCP, etc.), widely used, and built on a simple declarative language with extensive community support.

- **Statefile:** A JSON file (`terraform.tfstate`) that tracks what infrastructure Terraform is managing, the current attributes of each resource, and metadata. This is how Terraform knows what already exists vs. what needs to be created or changed. **Never manually edit or delete this file.**

- **Provider:** A plugin that enables Terraform to interact with a specific cloud platform or service. We use the Google provider to interact with GCP APIs.

- **HCL (HashiCorp Configuration Language):** Terraform's declarative configuration language. More human-readable than JSON. While primarily declarative, it includes some procedural features like loops and conditionals.

- **Idempotency:** A critical property of Terraform. Running `terraform apply` multiple times with the same configuration produces the same result. Terraform will not recreate or modify resources unless your code changes or state drift is detected.

- **Resource:** A block of HCL code that defines infrastructure to create or manage in the cloud, such as a VPC or VM instance. Each resource has a type and configuration parameters.

- **Execution Plan:** Generated by `terraform plan`, this shows exactly what Terraform will do before it does it. what resources will be created, modified, or destroyed, and in what order. This is Terraform's dry-run feature.

- **State Drift:** When actual infrastructure in GCP differs from what is recorded in the state file. This can happen if someone manually changes resources in the console. Terraform can detect and correct drift using `terraform plan` and `terraform apply`, though severe drift can exceed its built-in correction capabilities.

---

## Terraform Workflow

We will discuss this in more detail later, but these are the most important Terraform commands:

```bash
# Initialize the working directory: downloads provider plugins, generates lock file, etc.
terraform init

# Validate HCL syntax and configuration (checks "grammar", not semantics)
# This confirms Terraform understands your code but does not guarantee GCP will accept it
terraform validate

# Generate the execution plan: preview what will change and catch potential API-level errors
# May collect data from GCP but will never make changes
terraform plan

# Apply changes: actually create, modify, or destroy infrastructure
terraform apply

# Destroy all resources managed by this state file
terraform destroy
```

**Typical workflow:**

1. Write or modify `.tf` files
2. Run `terraform validate` to check syntax
3. Run `terraform plan` to preview changes
4. Review the plan carefully
5. Run `terraform apply` to execute
6. Terraform updates the state file automatically

**CLI documentation:** https://developer.hashicorp.com/terraform/cli/commands

---

## Development Environment Setup

Before we dive into Terraform, let's do some initial setup. We will be using the TheoWAF folder, specifically the Terraform subdirectory. We will also be working in VS Code. If you do not have Git Bash (as an administrator) or Terminal open, go ahead and open it now.

There are three commands everyone should be familiar with. These are the bread and butter of using the CLI:

- `pwd` - print working directory
- `cd` - change directory
- `ls` - list

### pwd

This stands for "print working directory." In IT, "print" simply means to output something to the screen. So `pwd` tells you the folder you are currently working in, expressed as an absolute file path.

A **file path** is the location of a file or directory on your computer, like a URL but for your local filesystem. There are two types:

**Absolute file path** - directions from a fixed starting point (the drive root) all the way to your destination:

```
/c/class/class-7.5/notes/week-4
```

This says: start at the `C:` drive, go into `class`, then `class-7.5`, then `notes`, then `week-4`.

**Relative file path** - directions from your current location rather than from the root. If you are already in `/c/class/class-7.5`, you can navigate to `week-4` with just `notes/week-4` instead of the full path.

This [video](https://www.youtube.com/watch?v=ephId3mYu9o) may help, but honestly just running these commands a few times is the fastest way to get comfortable with them.

### cd

`cd` (change directory) moves you into another directory. Use it with a path: `cd dir1/dir2` opens `dir1` and then moves into `dir2`.

### ls

`ls` lists the files and folders inside your current directory.

### Now You Try

In your CLI (Git Bash or Terminal), run `pwd` and `ls` and note the output. Then navigate to the correct folder:

```bash
cd $HOME/Documents/TheoWAF/class7.5/GCP/Terraform
```

Run `pwd` and `ls` again. You should see a `.json` file, a `.gitignore`, and a directory named `terraform-<something>` created by the `prelim.sh` script. Move into that directory:

```bash
cd <that directory's name>
```

Run `ls` and `pwd` again to confirm you see copies of the `.json` and `.gitignore` files. Finally, open VS Code from this location:

```bash
code .
```

---

## Exploring VS Code

We will start by creating a file in VS Code named `main.tf`. As far as Terraform is concerned, file names do not matter as long as they end in `.tf`- Terraform is declarative, so it does not care about order or file names, only about what you have declared.

Make sure the **Explorer** is open in the left pane. If it is not, click the explorer icon. Then create a new file using the "New File" button as shown [here](../assets/make-new-file.PNG).

Next, open an integrated terminal via the menu bar: **Terminal > New Terminal**. Mac users can use the default terminal. Windows users will see PowerShell open by default. Switch to Git Bash by clicking the **Launch Profile** chevron (the downward arrow), selecting **Select Default Profile**, then choosing **Git Bash** from the command palette. Click the **+** icon in the terminal panel to open a Git Bash session. It will default to Git Bash going forward.

At this point VS Code should look like [this](../assets/vs-code-done.PNG). You can close anything else except a browser with the GCP console open.

---

## Basic Terraform

### Basic Syntax

Before we deploy anything to GCP, let's look at what the language looks like and practice the workflow locally. This section does not require authentication since we are not touching GCP.

Terraform code is organized into **blocks**. A block defines what a section of code is for and usually gives it a unique **label**. A block contains **arguments** which are key-value pairs that configure the block. Arguments are wrapped in curly braces. The general structure:

```hcl
<block type> <label> {
  argument1 = value1
  argument2 = value2
}
```

Indentation is not required by Terraform, but the two-space indent shown above is the idiomatic style recommended by HashiCorp.

[Blocks and arguments docs](https://developer.hashicorp.com/terraform/language/syntax/configuration)

### The Terraform Block

All Terraform configurations should include a `terraform` block. This is largely boilerplate but lets you set the minimum required Terraform version. Add this to `main.tf`:

```hcl
terraform {
  required_version = ">= 1.5"
}
```

This block has no label and one argument, specifying that the configuration requires Terraform 1.5 or higher. Version 1.5 was the last release with major language changes, which is why it is a reasonable floor.

[Terraform block](https://developer.hashicorp.com/terraform/language/block/terraform)
[Version constraints](https://developer.hashicorp.com/terraform/language/expressions/version-constraints)

### Output

On its own, the `terraform` block does nothing visible. Let's add an `output` block, which prints a value to the terminal when Terraform runs:

```hcl
output "sample_label" {
  value       = "Hello, world!"
  description = "This is some working Terraform code!"
}
```

This block has one label (`sample_label`), a `value` argument (what gets printed), and a `description` argument (a note for us). Add this to `main.tf` after a blank line.

[Output docs](https://developer.hashicorp.com/terraform/language/block/output)

### Running the Workflow

1. `terraform init`- only needs to be run once per project (or when the `terraform` block changes). You should see a green success message.
2. `terraform validate`- confirms your syntax is correct.
3. `terraform fmt`- fixes formatting in place. If no changes are needed, it outputs nothing.
4. `terraform plan`- generates the execution plan (the "diff"). Ignore the note at the end; treat it as successful unless an explicit error is shown.
5. `terraform apply`- runs the code. Type `yes` to confirm. You should see your output message in the results as shown [here](../assets/output.PNG).

---

## The Google Provider and a Basic Infrastructure Example

The previous example skipped a major Terraform concept: the **provider**. A provider is a plugin that tells Terraform how to talk to a specific API. This section covers the Google provider and deploying a VM to GCP.

### Authentication

Terraform itself does not authenticate to GCP. The Google provider plugin handles that. There are three ways it can prove your identity to GCP, checked in this order:

1. **Provider credentials argument**- uses a service account JSON key file (we use this)
2. **Environment variables**- equally acceptable, slightly more setup required
3. **Application Default Credentials (ADC)**- uses your personal Gmail identity; least preferred

All setup for option 1 was completed during the install process. GCP generated a JSON key file for your account, which you should see in VS Code right now.

[Google Provider Authentication Docs](https://registry.terraform.io/providers/hashicorp/google/latest/docs/guides/provider_reference#authentication)

### Declaring the Google Provider

Create a new file called `auth.tf`. Move the `terraform` block from `main.tf` into it, and delete the `output` block from `main.tf`. Then update the `terraform` block in `auth.tf` to declare the Google provider as a dependency:

```hcl
terraform {
  required_version = ">= 1.5"

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 7.0"
    }
  }
}
```

This tells Terraform to download the Google provider and use the latest 7.x release. Next, add a `provider` block below it in `auth.tf`:

```hcl
provider "google" {
  project     = "YOUR_PROJECT_ID"
  region      = "YOUR_REGION"
  zone        = "YOUR_ZONE"
  credentials = "YOUR_JSON_FILENAME"
}
```

For example:

```hcl
provider "google" {
  project     = "seir-1"
  region      = "us-east1"
  zone        = "us-east1-b"
  credentials = "032326-tf-key.json"
}
```

Your project ID is shown on the GCP console welcome page and can be copied from there. Your JSON key filename will be different. Most errors at this step come from a typo here. Your `auth.tf` should look something like [this](../assets/auth.PNG).

### Writing the VM Resource

To create, modify, or delete something in GCP with Terraform, we use a `resource` block. A resource block takes two labels: one identifying the resource type (which tells the Google provider what to create) and one that is our internal name for the block in our code (not the resource's name in GCP). Add the following to `main.tf`:

```hcl
resource "google_compute_instance" "our_first_terraform_vm" {
  name         = "terraform-server"
  machine_type = "e2-medium"

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-12"
    }
  }

  network_interface {
    network = "default"

    access_config {} # External IP
  }
}
```

Breaking this down:

- `google_compute_instance` tells the Google provider to create a VM
- `our_first_terraform_vm` is what we call this block in our code
- `name` and `machine_type` are simple arguments
- `boot_disk` is a nested block that defines the persistent disk; `initialize_params` sets its initial configuration and `image` specifies the OS
- `network_interface` puts the VM in the default VPC; since it is the default VPC, no subnet argument is needed
- `access_config {}` with no arguments tells GCP to assign an external IP with default settings
- The `#` at the end of that line starts a comment, which Terraform ignores

No region or zone is specified in this resource block. When omitted, the Google provider uses the values set in the `provider` block in `auth.tf`.

[Terraform registry page on Google VM resource](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_instance)

### Running the Workflow

1. `terraform init`- required again since we added a provider. This downloads the Google provider binary into a `.terraform` directory and creates a `.terraform.lock.hcl` file recording which provider version was used.
2. `terraform validate`- confirm syntax is correct.
3. `terraform fmt`- fix any formatting if needed.
4. `terraform plan`- review the execution plan carefully before proceeding.
5. `terraform apply`- deploy the VM. Type `yes` to confirm.
6. Check the GCP console to verify your VM exists.
7. Run `terraform destroy` to tear it down, and confirm it is gone in the console.

Note: when `terraform apply` runs for the first time it creates `terraform.tfstate`. Do not delete or manually edit this file. It is how Terraform tracks everything it manages. Also avoid making changes to Terraform-managed resources via the console or `gcloud`, as this introduces state drift which is a mismatch between the actual state of your account and what the state file believes is there. On subsequent applies or the first `terraform destroy`, Terraform also creates `terraform.tfstate.backup`, a copy of the state file from before that operation ran.

---

## Learning Resources

- [Terraform GCP Tutorial](https://developer.hashicorp.com/terraform/tutorials/gcp-get-started)
- [HCL Reference](https://developer.hashicorp.com/terraform/language)
- [Google Provider Registry](https://registry.terraform.io/providers/hashicorp/google/latest/docs)