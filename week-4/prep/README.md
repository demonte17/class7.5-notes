# Class 7.5 / SEIR-1: Week 4 Prep Notes

## Table of Contents

---

## Introduction

This week will be covering the fundamentals of Terraform. This is an automation tool for provisioning and managing cloud infrastructure and other tools. It is a tool in the category called IaC (Infrastructure as Code). It is the most popular IaC tool (by far) and is cloud agnostic (meaning it works in essentially any public cloud and even private clouds).

I will start from the ground up by showing why we need this tool, issues it fixes, how to set it up, and eventually how to redo the week 1/2 labs with Terraform from scratch.

### Class Overview

* Introduction
* Discuss prelims
* Discuss goals
* VS Code, HashiCorp docs
* Discuss the end goal
* Discuss Terraform features
* Discuss HCL syntax
* Build out code
* Terraform workflow
* Troubleshooting

### Preliminary Materials

The script checks the following:

* Google Cloud SDK (in particular, `gcloud`) is installed and authenticated
* Terraform binary is installed and up to date
* TheoWAF folder present at `~/Documents/TheoWAF/class7.5/GCP/Terraform`
* Creates a `.gitignore` file
* Verifies a JSON token is present in the `Terraform` directory

It will create the TheoWAF folder structure if needed and will download a `.gitignore` file configured for Terraform projects.

```bash
curl https://raw.githubusercontent.com/aaron-dm-mcdonald/class7.5-notes/refs/heads/main/week-4/prep/scripts/prelim.sh | sh
```

---

## Why is this tool needed?

In enterprise environments automation becomes critical and is one of the main tasks of DevOps. Using the cloud how we have been thus far is called ClickOps (since we click in the console to do things). Cloud infrastructure allows for a great deal of automation so these two areas have a lot of crossover. Naturally automating the creation of infrastructure in the cloud platform follows. However there are other reasons to use this tool which become clear from the cons of ClickOps.

**Cons of ClickOps:**

* Difficult to reproduce across environments (dev, staging, prod)
* Not self-documenting: no record of what you clicked or why
* Impossible to automate or version control
* No single source of truth that is auditable

However *why* specifically this tool? Why is it special? First we should discuss more traditional means of automation. We will use the example of the week one lab (except to keep things short I will leave off the startup script).

---

## API

An API (Application Programming Interface) is at the heart of GCP and all cloud platforms. It allows programs, services, and computers to communicate with GCP. Every "button" in the GCP console actually makes API calls behind the scenes.

**API call example:**

Here we look at an HTTP request that could be sent to the GCP API. Let's break this down. Realistically this would never be done.

* A POST request is the opposite of what you normally do when you load a website's homepage. Instead of getting data, you are sending data.
* API endpoint is just a very specific URL (website address basically) that is meant to receive and then do certain things. Here we are using the Compute Engine endpoint.
* Host is the actual website or API location on the internet
* `HTTP/1.1` is the version of the HTTP protocol and format
* Content type is saying the "payload" (like the body of an email) is formatted as JSON
* Content length is how large in bytes this request is
* Authorization is where you would prove you are allowed to do something in GCP. How you generate this is a whole different discussion

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

* Extremely verbose and error-prone
* Requires manual authentication handling
* Not designed for human interaction
* No state tracking: you have to manually track what you've created
* Hard to make small edits

**Reference:** https://docs.cloud.google.com/compute/docs/reference/rest/v1/instances/insert

---

## Google Cloud SDK

The Google Cloud SDK (software development kit) is a collection of tools and libraries for engineers to use GCP. Of this you have heard of one specific tool already called `gcloud`. `gcloud` is the largest tool in this collection; however, there are many specialized tools and additionally there are libraries and modules for programming languages to work with GCP. We will ignore these extra tools for now and focus in on `gcloud`.

`gcloud` is a CLI utility that is written in Python. It allows us to interact with GCP simply by using various `gcloud` commands. It handles authentication for us when we run `gcloud init` by making credentials in a special folder on your computer via SSO (when you had to run `gcloud init` and then your web browser took you somewhere). It allows us to do almost everything we can do in the console (and what it can't do there are specialized, even better tools in the SDK). In fact, for the GCP services it's meant to manage, it can do *more* than what we can do in the console as far as certain features and configuration goes. It also lets us preset our project and set default region/zone. Instead of having to do that very verbose API call from above we could accomplish the same thing with:

```bash
gcloud compute instances create <INSTANCE_NAME> \
  --project=<YOUR_PROJECT_ID> \
  --zone=<YOUR_ZONE> \
  --machine-type=e2-medium \
  --image-family=debian-11 \
  --image-project=debian-cloud \
  --network=default
```

**Cons of GCP CLI:**

* Many flags needed for the many parameters
* Different commands needed for create, update, and delete operations
* Easy to make mistakes
* Very hard to reproduce
* No state: CLI doesn't remember what resources it created
* No idempotency: running the same command twice creates duplicate resources
* Entirely imperative: you say what you want and in the order you want and it does it (a good and bad thing depending on use case)

**Pros of GCP CLI:**

* I don't want you to think this tool is bad or unneeded; it just isn't the ideal tool to automate the creation of infrastructure
* It is used for "one-time" issues
* It can quickly assist with admin issues like wrapping SSH sessions or restarting a fleet of VMs
* It can be used in shell scripts easily
* It can effectively query (search for) and filter (select the important) information from resources in GCP

---

## The Terraform Solution

Terraform solves these problems from the last few sections by:

* Using declarative code (describe what you want, not how to create it)
* Being stateful (it knows what exists and what needs to change)
* Being idempotent (safe to run multiple times if no configuration changes)
* Managing dependencies automatically (it solves the "dependency chain" by figuring out what needs to be made first, which is very good when you have say 50 things that need to be created)
* Supporting version control
* Enabling code reuse

### What is Terraform?

Terraform is an infrastructure as code (IaC) tool. Infrastructure as Code means that you write out everything as actual code just like a software engineer would. You test it along the way just like a software engineer would. You use each step of the software development lifecycle and methodology like a software engineer would, but rather than writing application code, you are writing code that defines infrastructure.

Terraform lets you define, provision, query, and manage cloud resources using a declarative configuration language. Instead of clicking through the GCP console or writing complex CLI commands, you write code that describes your desired infrastructure, and Terraform handles all the API calls and sequencing needed to make it happen.

**Official documentation:** https://developer.hashicorp.com/terraform/docs

### Key Terms

* **IaC (Infrastructure as Code):** The practice of managing infrastructure through code rather than manual coneednfiguration. Terraform is an IaC tool that allows us to write code that interacts with cloud provider APIs.

* **Terraform:** An open-source IaC tool created by HashiCorp for cloud automation. Cloud agnostic (works with AWS, Azure, GCP, etc.), widely used, simple declarative language, and extensive community support.

* **Statefile:** A JSON file (`terraform.tfstate`) that keeps track of what infrastructure Terraform is managing, the current attributes of each resource, and metadata. This is how Terraform knows what already exists vs. what needs to be created or changed. **Never manually edit or delete this file.**

* **Provider:** A plugin that enables Terraform to interact with a specific cloud platform or service. In our case, we use the Google provider to interact with GCP APIs.

* **HCL (HashiCorp Configuration Language):** Terraform's declarative configuration language. More human-readable than JSON. While primarily declarative (you describe what you want), it includes some procedural features like loops and conditionals.

* **Idempotency:** A critical property of Terraform; running `terraform apply` multiple times with the same configuration produces the same result. Terraform won't recreate or modify resources unless your code changes or state drift is detected.

* **Resource:** A block of HCL code that defines infrastructure or configuration to create or edit in the cloud, like a VPC or VM instance. Each resource has a type and configuration parameters.

* **Execution Plan:** Generated by `terraform plan`, this shows exactly what Terraform will do before it does it; what resources will be created, modified, or destroyed, and in what order. This is Terraform's "dry run" feature.

* **State Drift:** When the actual infrastructure in GCP differs from what's recorded in the Terraform state file. This can happen if someone manually changes resources in the console. Terraform can detect and correct drift using `terraform plan` and `terraform apply`; however, sometimes it exceeds the capabilities of the integrated drift correction tools.

---

## Terraform Workflow
We will discuss this more later but for now these are the most important terraform commands:


```bash
# Initialize working directory: downloads provider plugins, generates lock file, etc
terraform init

# Validate HCL syntax and configuration (check your "grammar", not if it "makes sense" otherwise known as semantics)
# This tests if Terraform understands your code but does not guarantee that GCP will
terraform validate

# Generate the execution plan and diff: preview what will change and catch some possible errors from the GCP API
# This may collect and save data from GCP but will never change anything
terraform plan

# Apply changes: actually create/modify/destroy infrastructure
terraform apply

# Destroy all resources managed by this statefile
terraform destroy
```

**Typical workflow:**

1. Write or modify `.tf` files
2. Run `terraform validate` to check syntax
3. Run `terraform plan` to see what will change
4. Review the plan carefully
5. Run `terraform apply` to make the changes
6. Terraform updates the state file automatically

**CLI documentation:** https://developer.hashicorp.com/terraform/cli/commands

---


## Get setup

Before we dive into Terraform lets do some inital steps. We will be using the TheoWAF folder (specifically the terraform subdirectory...I will call them directories from now on and so should you). Additionally we will be working with VS Code. If you don't have Git Bash (as an administrator) or Terminal open then go ahead and do so now. 

I need to introduce 3 commands everybody should be familar with. These are the bread and butter of using the CLI. 
- `pwd` - print working directory 
- `cd` - change directory 
- `ls` - list 

Lets look at these in more detail. 

### pwd 

So this stands for "print working directory" and that might sound complicated. We already know "directory" is like a folder. "Print" simply means, in IT terminology, to output something. So this command will output (or simply: tell us) our "working folder." That sounds less complicated. "working" in this context simply means "the directory you are currently using". So all together pwd will "tell us the folder we are using." How it does that is by telling us the absolute file path. Lets explain file paths. 

**file path** is the location (like a url but on your computer) of a file or directory. There are two kinds. 

**Absolute file path** is like getting directions from point A to point B. It doesn't matter where you are, you can still explain how to get to point A from point B. Point A for an absolute file path is the specific drive (like C:) and point B is the final file/directory you are interested in. It tells you each step. 

`/c/class/class-7.5/notes/week-4` is my current absolute file path. It says that if I were to start in the `C:` drive and go to the `class` directory and then go to the `class-7.5` directory and then the `notes` directory and then the `week-4` directory I would get to where I am "working" currently. From point A to Point B. 

**Relative file path** is worth mentioning here. It is the other type of file path. Instead of saying from point A to point B it assumes we are not at point A but somewhere along the directory and gives only the _needed_ directions. 

Say I am here: `/c/class/class-7.5` and wanted to get to the `week-4` folder, I could tell my computer to go to `/c/class/class-7.5/notes/week-4` and that would work fine, but I also could tell it "go to `notes/week-4` and it would see that I am in the `class-7.5` directory and then go into the `notes` directory and then into the `week-4` directory. 

This [video](https://www.youtube.com/watch?v=ephId3mYu9o) might help, but just using these commands will help the most. It sounds wordy on paper but it isn't a hard concept to understand. 

So in the end, `pwd` simply tells us the absolute file path of the directory we are currently in. 

### cd

This command (change directory) allows us to change our working directory. Essentially it lets us open other directories and move into them. we use it with a file path next to it like this: `cd dir1/dir2` would open the `dir1` folder and then the `dir2` directory. 

### ls 

This command just shows us the folders and files in the working directory. It lets us look inside the directory. 

### Now you do it!

In your CLI (git bash or terminal) run `pwd` and run `ls` and note what is there. On git bash `ls` may produce quite a bit of output. Lets go to the correct folder. Run `cd $HOME/Documents/TheoWAF/class7.5/GCP/Terraform` (note: this is an absoulute file path but I use a variable in here, don't worry about it). Now run `pwd` and `ls` and look at the output. You should see a file that ends in `.json` and a file called `.gitignore`. You should see a directory (it will end in `/` like all directories) called `terraform-040826/`. Let's move in that directory. Run `cd terraform-041026` and then run `ls` and `pwd` to see what changed. You should see a copy of that `.json` and `.gitignore` file in there. Finally lets open VS Code by running the command `code .` and VS Code will open. 











---
### Authentication

Terraform itself does not need to authenticate itself (prove who it is to GCP). However, the Google plugin that is used with Terraform will need some way to prove it is allowed to make API calls to your GCP account. There is a specific order to this:

1. **Provider credentials argument** — The ideal method that uses a service account and JSON token (we use this)
2. **Environment variables** — Equally acceptable method but requires slightly more setup
3. **ADCs** — This is the same as letting Terraform use your Gmail identity to run API calls and is the least ideal

This does not  to make a lot of sense right now. We will come back to the important part and you actually already did the legwork for this during software installs.

**Authentication documentation:** https://registry.terraform.io/providers/hashicorp/google/latest/docs/guides/provider_reference#authentication





## Learning Resources

Official Terraform documentation:

* https://developer.hashicorp.com/terraform/tutorials/aws-get-started
* https://developer.hashicorp.com/terraform/language
* https://registry.terraform.io/providers/hashicorp/aws/latest/docs
