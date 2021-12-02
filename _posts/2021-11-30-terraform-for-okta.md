---
layout: post
title: "Getting started with Terraform for Okta - Part 1"
date: 2021-11-30
---
#### Introduction
Getting started with using Terraform for Okta can be a bit confusing. There are [a small few](https://developer.okta.com/blog/2020/02/03/managing-multiple-okta-instances-with-terraform-cloud) tutorials online already but none really dealt with my specific environment of:

1. Having an existing and _complex_ Okta account already in existence.
2. Needing a remote state to collaborate with others in my team.
3. Wanting non-super admins to be able to contribute and `plan` their changes.

With these requirements in mind - let's take a look at how we migrated our existing Okta account to being mostly managed via Terraform.

In part 1 we will look at satisfying these requirements to create new resources and in the next post we will look at importing existing resources so they can be managed by Terraform instead.

#### Getting setup
As with most Terraform projects, we begin by creating our `main.tf` file to house our primary configuration info. In this file we will configure our Terraform provider and our remote state. In my case I am using AWS S3 as my remote state store - but Terraform [supports most major cloud providers](https://www.terraform.io/docs/language/state/remote.html). (Or if you're a one person team not interested in collaborating you can just keep a local state!)

```terraform
terraform {
  required_providers {
    okta = {
      source  = "okta/okta"
      version = "~> 3.20"
    }
  }

  backend "s3" {
    bucket = <insert_bucket_name_here>
    region = <insert_bucket_region_here>
    key = "resources/state"
  }
}

provider "okta" {
  org_name = <your_okta_org_subdomain>
  base_url = "okta.com"
}
```

Here, we are creating our configuration for Terraform using the Okta provider, and setting the required values to configure our remote state in AWS S3. 

__If you are also using S3 for the remote state please make sure you do not enable any public access to the bucket!__ When you have created the bucket in AWS - add the `bucket`, `region` and `key` values. You will notice that I am using `resource/state` for my key - this is related to separating SuperAdmin and non-SuperAdmin objects and I'll come back to that in a bit. You can explicitly set your AWS credentials in this file too if you want but by default Terraform will simply read your environment for an `AWS_PROFILE` variable and use that. 

###### _Credentials_
Terraform will look for the `AWS_PROFILE` and `OKTA_API_TOKEN` variables in your environment for authentication so it may be advisable to add both of these to a `.env` file in the root of this repository and use `source .env` in terminal to set those. Just make sure you add `.env` to your `.gitignore` file so they don't get accidentally commited to your Github repo.

Once you have got your `main.tf` file filled out with all the values for your environment, you can run `terraform init` in the root of your repo. Terraform will download the Okta provider and get itself configured.

#### Creating a new resource
Now that you have your base configuration done - we can create a new resource in Okta from Terraform. In this case let's create a new group and corresponding group rule directly from Terraform. You can choose what file structure works best for you - but in our case we have decided to pair groups and rules in the same `groups.tf` file as we rarely, if ever, create a group without a corresponding rule at our org. 

Create the `groups.tf` file in the same directory as your `main.tf` file. It is extremely useful to have the [official Okta Terraform Provider docs ](https://registry.terraform.io/providers/okta/okta/latest/docs) open as you work here - it details all the possible attributes that we can pass to both the [`okta_group`](https://registry.terraform.io/providers/okta/okta/latest/docs/resources/group) and [`okta_group_rule`](https://registry.terraform.io/providers/okta/okta/latest/docs/resources/group_rule) resources.

Let's create a group and rule called Engineering Managers. You'll notice we set `skip_users` to true. This tells Terraform that we won't be managing the membership of this group via Terraform, as we will be using a rule to do that. 

```terraform
resource "okta_group" "engineering_managers" {
	name		    = "Engineering Managers"
	description	= "A group for all Engineering Managers"
	skip_users	= true
}

resurce "okta_group_rule" "engineering_managers_group_rule" {
	name				      = "Engineering Managers - Group Rule"
	group_assignments	= okta_group.engineering_managers.id
	expression_type		= "urn:okta:expression:1.0"
	expression_value	= "String.stringContains(user.title, \"Engineering Manager\")"
	status            = "ACTIVE"
}
```

Here we have defined a group named "Engineering Managers", and a group rule named "Engineering Managers - Group Rule" that populates the former if any user's title contains the words "Engineering Manager". You can see that in the `group_assignments` attribute, rather than explicitly having to retrieve and set the group ID - we can use Terraform to refer to the `id` value directly from the `engineering_managers` resource! This prevents our rule and group linking from being fragile, and means if we ever need to delete and recreate them they will be linked by default. You also need to make sure you escape the quotation marks within your rule's `expression_value` so that Terraform doesn't try to interpret it literally. 

Now that we have written our group and rule into our configuration, we can run `terraform plan` to see what Terraform _would do_ if we applied these configurations:

```shell
Terraform used the selected providers to generate the following execution plan. Resource actions are indicated
with the following symbols:
  + create

Terraform will perform the following actions:

  # okta_group.engineering_managers will be created
  + resource "okta_group" "engineering_managers" {
      + description = "A group for all Engineering Managers"
      + id          = (known after apply)
      + name        = "Engineering Managers"
      + skip_users  = true
    }

  # okta_group_rule.engineering_managers_group_rule will be created
  + resource "okta_group_rule" "engineering_managers_group_rule" {
      + expression_type   = "urn:okta:expression:1.0"
      + expression_value  = "String.stringContains(user.title, \"Engineering Manager\")"
      + group_assignments = (known after apply)
      + id                = (known after apply)
      + name              = "Engineering Managers - Group Rule"
      + status            = "ACTIVE"
    }

Plan: 2 to add, 0 to change, 0 to destroy.
```

This looks good to me so next I will apply these changes with `terraform apply`. Here, again, Terraform will let you know what it _intends_ to do before explicitly asking you to type `yes` and hitting enter to apply the configuration. Once you've done this, and assuming you have sufficient permissions in Okta - Terraform will create the configured resources accordingly. 

This set of resources should now be managed by Terraform going forward to prevent drift from the configuration files (i.e. you should refrain from manual changes in the UI for this pair of resources). Existing groups not in Terraform can continue to be managed by the admin UI until they are imported (which we'll talk about in the next blog post.)

#### Collaborating and Democratisation
One of my main goals in moving our Okta infrastructure to Terraform was to allow non-Super Admins to contribute to resources that would otherwise be locked down, without incurring any additional risk. For example, in our org we don't allow our helpdesk/services team to directly modify groups or rules as the blast radius from a mistake to one of our larger groups here would be huge (think if our core group that assigns Google Workplace, HR tools, etc was accidentally modified - removing all memberships!), but they frequently have a strong need to modify or add new groups. By moving our infrastructure to Terraform the team can modify these plans and open a PR in Github to have their changes reviewed by another member of the team before being deployed.

The issue we ran into here is that it's impossible to run `terraform plan` without at least having read access to any resource you want to work with. We solved this by making all of our Okta admins Read Only administrators, on top of their write permissions. However, the other issue we encountered is that even Read Only admins can't view the Administrators section of the Okta dashboard - so if you want to maintain configuration files for your admins (and I really think you should for strong auditability) you will hit a blocker here. 

We solved for this by splitting our Terraform repo into two subdirectories. Our Terraform repo looks like this:

```shell
project_root
├── admins
	├── main.tf
	├── admins.tf
├── resources
	├── main.tf
	├── resources.tf
	├── <other .tf files>
```

The two `main.tf` files are identical except for `key` in the S3 configuration. As alluded to earlier - in the `resource/` folder we use the `resources/state` key, and in the `admins/` directory we use the `admins/state` key. These could also be something like `state_admins` and `state_resources` - it doesn't really matter so long as they're distinct.

#### Conclusion
In this first part of our look at Terraform and Okta - we have created our basic configuration and created a pair of brand new resources in Okta. We have also talked about how we can empower non-SuperAdmins to contribute to our infrastructure without having full write permissions to sensitive elements within Okta. In the next post we will look at how we tackled importing existing resources from our Okta account into Terraform configurations and state. 

If you have any questions in the meantime come hit me up in the [MacAdmins Slack](https://www.macadmins.org/) - [@tim.fitzgerald](https://macadmins.slack.com/team/U0HV0URL7).
