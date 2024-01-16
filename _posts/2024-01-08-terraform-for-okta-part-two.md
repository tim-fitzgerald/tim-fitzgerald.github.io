---
layout: post
title: "Terraform for Okta - 2 years on."
date: 2024-01-15
---
I always intended to write up a Part 2 of my [Getting Started With Terraform for Okta](https://me.timfitzgerald.io/2021/terraform-for-okta/) post. This part is less of a guide like Part 1, and instead this Part 2 is more of a "What have I learned" affair.

#### OAuth
The biggest change since part one is that we no longer use a long lived Okta API token to plan and apply Terraform changes. Something about keeping long lived admin tokens on disk started to make me a tad uneasy, so we migrated to using OAuth via an API _application_ in Okta. The resources on how to do this are [pretty well documented](https://developer.okta.com/docs/guides/terraform-enable-org-access/-/main/#add-credentials-to-terraform) but I do want to highlight a couple of design choices I made, and some oddities I ran into. 
- Use a separate API application for your plans and your applies - especially if you want to ensure all members of your IT can run a plan, but only a smaller group can apply. You can ensure that the `plan` application only has `read` scopes, whereas the `apply` application should have `read` and `manage` scopes.
- I thought initially I could just assign a Read-Only Admin role to the `plan` application, but for some undocumented reason this role doesn't have permission to read custom application schema (very very weird). My workaround here was to also give it the Application Administrator role as well (could not find a custom resource set that worked, seems to be undocumented privileges but please get in touch if you have found otherwise). This is still mostly ok because the app is still restricted by its scopes, but it is a more powerful admin role than I ideally wanted to assign.
- Similarly to the last point, the built in Read-Only Admin role could not read admin roles themselves, so I couldn't run plans that included admin user definitions. I was able to solve this one using a custom Admin role that has Read-Only rights on `All Identity and Access Management Resources`. Previously we had separate Terraform repos for our admin resources and everything else, which meant that non Super Admins couldn't audit and validate our admin resources - but this is now resolved and we can combine the two repos into one. This will keep your auditors happy. 
- Despite what the UI may indicate - non Super Admin users _cannot_ generate new key pairs for the OAuth application. So that's good. (At first, the UI kind of indicated that regular admins _could_ do this which would allow for self-escalation into the `apply` OAuth application).
And while are on the topic of key pairs and credentials - lets talk about how manage those.

#### 1Password Secret References
While investigating a good method for getting long lived credentials out off `.env` files stored in disk, I discovered a neat new(ish) feature of 1Password called [Secret References](https://developer.1password.com/docs/cli/secret-references/). This allows you to use 1Password CLI to decrypt secrets from the 1Password application on the fly, and inject them directly into commands as environment variables in realtime by acting as a command runner/wrapper.

In our case we store the private key as a 1Password secure note (storing it as a credential messes up the formatting and it doesn't work) and then use `op item get <item_name> --format json` to get a `reference` address. We then export that address that as the `OKTA_API_PRIVATE_KEY` environment variable value. When it comes time to run a Terraform plan you use the 1Password CLI to wrap the command, and it will dynamically inject the _real_ secret on the fly, e.g.:

```shell
export OKTA_API_PRIVATE_KEY="op://some-account/some-vault/some-item"
op run -- terraform plan
# The OP CLI tool now retrieves the item at that address, and passes the real value into `terraform plan`
```

Leveraging this approach we can centrally store our `plan` credentials in a team-wide shared vault, and restrict our `apply` credentials to a smaller shared vault as is applicable.

#### Handling Environments
We just spoke about how we handle different _secrets_ for different environments, but we haven't spoken about variables. As we have two separate API OAuth applications, we have two completely different sets of details for each (e.g. `Client ID`, scopes, etc). These aren't considered secrets and should not be obscured from anyone within the organisation so we handle these as native Terraform variables. We achieve this using a combination of `.tfvars` files and a `variables.tf` file. We define the variables in our `variables.tf` file with default values if we wish. (Note that these are shortened for brevity but you will also need to define variables for the `private_key_id` and `client_id`.)

```terraform
variable "okta_scopes" {
  type = list(string)
  default = [
  "okta.groups.read",
  "okta.roles.read",
  "okta.schemas.read",
  "okta.users.read"
  ]
}
```

Our `.tfvars` files can then specify our different values for that environment - for example, your `apply.tfvars` file may look like this:

```terraform
okta_scopes = [
"okta.groups.read",
"okta.groups.manage",
"okta.roles.read",
"okta.roles.manage",
"okta.schemas.read",
"okta.schemas.manage",
"okta.users.read",
"okta.users.manage"
]
```

Finally in our `main.tf` file we set up the `scopes` attribute to read from this variable like so:

```terraform
provider "okta" {
  org_name       = "<your_okta_subdomain>"
  base_url       = "okta.com"
  scopes         = var.okta_scopes
  client_id      = var.client_id
  private_key_id = var.okta_private_key_id
}
```
Note that the default is to only use `read` scopes and we override that with `manage` scopes only when needed. We specify which `tfvars` file we want to use when we call `terraform plan` or `terraform apply` by passing in the flag `-var-file=<path_to_file.tfvars>`

#### Automating the repetition
This is all very repeatable and secure, but its really not user friendly. Given that we are leveraging these Secret References, and storing our state in S3 (behind Okta), to run a plan for any member of the team would require the following:

1. Get a valid set of AWS credentials (we use `okta-aws-cli`).
2. Make sure you set your correct AWS role via the `AWS_PROFILE` env var.
3. Make sure you set the `OKTA_API_PRIVATE_KEY` env var.
4. Make sure you set the `OKTA_API_PRIVATE_KEY_ID` env var.
5. Run `op run -- terraform plan -var-file=plan.tfvars` or `op run -- terraform plan -var-file=applys.tfvars`

Its not the end of the world, but it's a bit too much manual work. And I didn't get to this point in my career by doing manual work. At Clio we use an internally developed command runner called `dev`, but the following principle could apply to any command runner such as [just](https://github.com/casey/just), or even just plain ol' bash scripts turned executables. Regardless of what you use, you want one single command to run all of the above so that your users don't need to repeat several commands every time they want to run a plan, and don't need to maintain their own `.env` files - as we want to be able to handle credential rotation automatically via our 1Password item.

#### Notes
- If you are so inclined, you can make your `terraform apply` run in Github Actions - but I'll be honest that as of this moment that still makes me uneasy. I still want to know that a human is running the `apply`, giving the plan one last review, and explicitly typing `yes` to the prompt. 
- Depending on the size of your Okta tenant, you will hit API rate limits running `plan` and `apply` to varying degrees. You can work around this by not using a monorepo, all the same logic / tips would apply you would just create separate repositories for each group of resources. This is on our to-do list.


