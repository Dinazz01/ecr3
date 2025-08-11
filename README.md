# ecr3
dd a toggle

variable "enable_signer" {
  description = "Create AWS Signer profile if true; reuse existing if false"
  type        = bool
  default     = false
}

variable "existing_signing_profile_arn" {
  description = "ARN of existing profile to reuse when enable_signer=false"
  type        = string
  default     = ""
}
2. Wrap resource in conditional count
h
Copy
Edit
resource "aws_signer_signing_profile" "container" {
  count       = var.enable_signer ? 1 : 0
  name        = local.signing_profile_name
  platform_id = var.signer_platform_id

  lifecycle {
    prevent_destroy = true
  }
}
3. Build a “works in both modes” local
hcl
Copy
Edit
locals {
  signing_profile_name = substr(
    replace("ecr-container-signing-${var.env_abbr}-${var.name}", "/[^a-zA-Z0-9]/", ""),
    0,
    64
  )

  signing_profile_arn = var.enable_signer
    ? aws_signer_signing_profile.container[0].arn
    : var.existing_signing_profile_arn
}
4. Use the local everywhere instead of hardcoding
Wherever you reference the signer profile ARN, swap in:


local.signing_profile_arn
5. First-time creation vs. reuse
First run:
enable_signer = true
Terraform creates the profile, stores it in state, and will not try to recreate it next time.

Reuse existing:
enable_signer = false and set existing_signing_profile_arn to the ARN of the existing profile.

Import existing into state (if it’s in AWS but not in Terraform yet):


terraform import aws_signer_signing_profile.container[0] signing-profile/<existing-name>
If you want, I can rewrite your current locals block from the screenshot so it has this toggle logic built in and won’t blow up when the profile already exists. That will fix the signer error without touching your EKS and KMS blocks.








You said:
KMS key policy has an invalid ARN
Error: InvalidArnException: An ARN in the specified key policy is invalid.
Common causes:

Using a role/user name instead of a full ARN.

Building the ARN with the wrong partition or a blank account id (ends up like arn:aws:iam:: :role/...).

Including wildcards where ARNs are required.





Yep—that error almost always comes from how the ARNs are built. Here’s a tight, safe pattern you can drop in to eliminate InvalidArnException for your KMS key policy.

1) Get the right account & partition (works in Gov/China too)

data "aws_caller_identity" "current" {}
data "aws_partition" "current" {}
2) Turn role names into full ARNs (never paste names directly)

variable "kms_user_role_names" {
  description = "IAM role names that may use the KMS key (same-account)."
  type        = list(string)
  default     = []
}

variable "extra_principal_arns" {
  description = "Any additional, already-complete ARNs (cross-account, users, etc.)."
  type        = list(string)
  default     = []
}

locals {
  account_id = data.aws_caller_identity.current.account_id
  part       = data.aws_partition.current.partition

  role_arns = [
    for n in var.kms_user_role_names :
    "arn:${local.part}:iam::${local.account_id}:role/${n}"
  ]

  # final principals list (no blanks)
  principals = compact(concat(
    ["arn:${local.part}:iam::${local.account_id}:root"],
    local.role_arns,
    var.extra_principal_arns
  ))
}
3) Key with a bullet-proof policy
h
Copy
Edit
resource "aws_kms_key" "ecr" {
  description         = "CMK for ECR"
  enable_key_rotation = true

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid      = "EnableRootPermissions"
        Effect   = "Allow"
        Principal = { AWS = "arn:${local.part}:iam::${local.account_id}:root" }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "AllowRolesToUseKey"
        Effect = "Allow"
        Principal = { AWS = local.principals }
        Action = [
          "kms:Encrypt","kms:Decrypt","kms:ReEncrypt*",
          "kms:GenerateDataKey*","kms:DescribeKey"
        ]
        Resource = "*"
      }
    ]
  })

  # Guardrails: catch bad/empty ARNs early
  lifecycle { prevent_destroy = true }

  depends_on = [] # keep if you compute principals elsewhere
}
4) Validate inputs (optional but handy)
hcl
Copy
Edit
variable "extra_principal_arns" {
  type = list(string)
  default = []
  validation {
    condition = alltrue([
      for a in var.extra_principal_arns :
      length(regexall("^arn:${data.aws_partition.current.partition}:iam::[0-9]{12}:(role/.+|user/.+|root)$", a)) > 0
    ])
    error_message = "extra_principal_arns must be valid IAM ARNs (role/..., user/... or root)."
  }
}
5) Checklist—most common fixes
Use full ARNs, never plain names.

Don’t hand-type arn:aws:... if you might be in Gov/China—use data.aws_partition.current.partition.

Ensure account_id is not blank (don’t use try(...,"") for it).

Don’t pass STS assumed-role ARNs (arn:aws:sts::...:assumed-role/...) in key policies; use the IAM role ARN (arn:aws:iam::...:role/...).

Remove empty strings from principal lists (compact(...)).

For cross-account, include the other account’s role/user ARN in extra_principal_arns.

If you paste your current policy = jsonencode({ ... }) block (or the vars you feed into it), I’ll pinpoint the exact ARN that’s invalid and correct it.








You said:

pulling from an exinting KMS module 



an existing KMS module and the key policy is blowing up. From your screenshot, the issue is almost certainly the inputs you pass into the module, not the module itself.

Here are the exact fixes:

Don’t pass empty strings as principals
coalescelist(..., [try(..., "")]) can produce [""], which makes KMS throw InvalidArnException. Always wrap principal lists with compact(...).

Normalize role names → full ARNs
If local.repository_role or any “user/owner/admin” entries are names, convert to arn:...:iam::ACCOUNT_ID:role/NAME.

Avoid STS ARNs
KMS key policies must reference IAM ARNs (arn:aws:iam::...:role/...), not STS assumed-role ARNs. If you use aws_iam_session_context.current[0].issuer_arn, that’s usually an IAM role ARN (good), but still guard for null/empty.

(From your shot) You’ve got key_administrators defined twice
Terraform will complain or the later one wins. Keep only one.




data "aws_caller_identity" "current" {}
data "aws_partition" "current" {}

locals {
  part       = data.aws_partition.current.partition
  account_id = data.aws_caller_identity.current.account_id

  # If you have role *names*, convert to ARNs
  repository_role_arn = (
    can(regex("^arn:${local.part}:iam::[0-9]{12}:role/", local.repository_role))
    ? local.repository_role
    : "arn:${local.part}:iam::${local.account_id}:role/${local.repository_role}"
  )

  # Safe “maybe” current role (issuer_arn is IAM role when present)
  maybe_current_issuer = try(data.aws_iam_session_context.current[0].issuer_arn, "")

  # Clean lists (remove blanks) and ensure ARNs
  key_admins = compact(coalescelist(
    var.kms_key_administrators,       # expect ARNs here if you already pass ARNs
    [local.maybe_current_issuer]      # may be "", compact() removes it
  ))

  key_users = compact(concat(
    # if you pass names here, convert them first (or pass ARNs directly)
    var.kms_key_users,                # prefer ARNs
    [local.repository_role_arn]
  ))

  key_owners = compact(var.kms_key_owners)
  svc_users  = compact(var.kms_key_service_users)
}

module "kms" {
  source  = "terraform.highmark.com/hm-shared/aws-kms/aws"
  version = ">= 1.0.0" # align with your registry

  count = var.create_kms_key ? 1 : 0

  description         = coalesce(var.kms_key_description, "${var.repository_name} repository encryption key")
  key_usage           = "ENCRYPT_DECRYPT"
  deletion_window_in_days = var.kms_key_deletion_window_in_days
  enable_key_rotation = var.kms_key_rotation

  # Policy inputs
  enable_default_policy   = var.kms_key_enable_default_policy
  key_owners              = local.key_owners
  key_administrators      = local.key_admins
  key_users               = local.key_users
  service_users           = local.svc_users
  source_policy_documents = var.kms_key_source_policy_documents
  override_policy_documents = var.kms_key_override_policy_documents

  aliases = var.kms_key_aliases
}
Optional input validation (helps catch mistakes early):


variable "kms_key_users" {
  type = list(string)
  default = []
  validation {
    condition = alltrue([
      for a in var.kms_key_users :
      length(regexall("^arn:${data.aws_partition.current.partition}:iam::[0-9]{12}:(role|user)/.+$", a)) > 0
    ])
    error_message = "kms_key_users must be IAM ARNs (role/... or user/...)."
  }
}

variable "kms_key_administrators" {
  type = list(string)
  default = []
  validation {
    condition = alltrue([
      for a in var.kms_key_administrators :
      length(regexall("^arn:${data.aws_partition.current.partition}:iam::[0-9]{12}:(role|user)/.+$", a)) > 0
    ])
    error_message = "kms_key_administrators must be IAM ARNs."
  }
}
Quick checklist for your module inputs:

Ensure every principal list you send (key_users, key_administrators, key_owners, service_users) has only valid IAM ARNs and no empty strings → wrap with compact(...).

Convert any names to ARNs using arn:${partition}:iam::${account_id}:role/${name}.

Remove the duplicate key_administrators attribute in the module block.

If you need to include the “current executor” as an admin, keep issuer_arn but only after compact(...).

If you paste your actual values for var.kms_key_* and local.repository_role, I’ll convert them to the exact ARNs you need and verify the module call end-to-end.








 details example for this recommendations                                                                                                                                                                           Ensure every principal list you send (key_users, key_administrators, key_owners, service_users) has only valid IAM ARNs and no empty strings → wrap with compact(...).

Convert any names to ARNs using arn:${partition}:iam::${account_id}:role/${name}.

Remove the duplicate key_administrators attribute in the module block.

If you need to include the “current executor” as an admin, keep issuer_arn but only after compact(...).



#concrete Terraform examples for each point, with “before/after” patterns you can copy-paste.

1) Ensure principal lists contain only valid IAM ARNs (and no empty strings)
❌ Before (can inject empty strings, names, or STS ARNs)

variable "key_users" { type = list(string) }
variable "key_admins" { type = list(string) }

locals {
  # Might be names or blank strings coming from vars
  raw_key_users  = var.key_users
  raw_key_admins = concat(var.key_admins, [try(data.aws_iam_session_context.current[0].issuer_arn, "")])
}

module "kms" {
  # ...
  key_users          = local.raw_key_users      # BAD: may contain names / blanks
  key_administrators = local.raw_key_admins     # BAD: may contain ""
}
✅ After (validated ARNs + blanks removed)

data "aws_caller_identity" "current" {}
data "aws_partition" "current" {}

variable "key_users"  { type = list(string) default = [] } # expect ARNs or names (we’ll normalize)
variable "key_admins" { type = list(string) default = [] }

locals {
  part       = data.aws_partition.current.partition
  account_id = data.aws_caller_identity.current.account_id

  # Normalize possibly mixed inputs (names or ARNs) → ARNs
  norm = {
    users  = [for u in var.key_users  : can(regex("^arn:", u)) ? u : "arn:${local.part}:iam::${local.account_id}:role/${u}"]
    admins = [for a in var.key_admins : can(regex("^arn:", a)) ? a : "arn:${local.part}:iam::${local.account_id}:role/${a}"]
  }

  # Remove any accidental blanks
  key_users_arns  = compact(local.norm.users)
  key_admins_arns = compact(local.norm.admins)
}

module "kms" {
  # ...
  key_users          = local.key_users_arns
  key_administrators = local.key_admins_arns
}
Optional input validation to catch bad ARNs early:

hcl
Copy
Edit
variable "key_users" {
  type    = list(string)
  default = []
  validation {
    condition = alltrue([
      for a in var.key_users :
      can(regex("^arn:${data.aws_partition.current.partition}:iam::[0-9]{12}:(role|user)/.+$", a)) || !can(regex("^arn:", a))
    ])
    error_message = "If you pass ARNs in key_users, they must be IAM ARNs (role/... or user/...). Names are also allowed and will be converted."
  }
}
2) Convert names to ARNs using arn:${partition}:iam::${account_id}:role/${name}
Example: mixed list (some names, some ARNs)
hcl
Copy
Edit
# Input (could come from tfvars)
# key_users = ["deploy-role", "arn:aws:iam::123456789012:role/reader"]

locals {
  part       = data.aws_partition.current.partition
  account_id = data.aws_caller_identity.current.account_id

  key_users_arns = [
    for id in var.key_users :
    can(regex("^arn:", id)) ? id : "arn:${local.part}:iam::${local.account_id}:role/${id}"
  ]
}
3) Remove the duplicate key_administrators attribute in the module block
❌ Before (duplicate attribute; last wins, confusing)
h
Copy
Edit
module "kms" {
  source = "..."
  # ...
  key_administrators = var.kms_key_administrators
  key_administrators = concat(local.repo_admins, var.kms_key_users) # duplicate!
}
✅ After (single, explicit attribute)
hcl
Copy
Edit
locals {
  final_key_admins = concat(local.repo_admins, var.kms_key_administrators)
}

module "kms" {
  source = "..."
  # ...
  key_administrators = local.final_key_admins
}
4) Include the “current executor” as an admin using issuer_arn, but only after compact(...)
❌ Before (can introduce empty string)
hcl
Copy
Edit
data "aws_iam_session_context" "current" { count = 1 }

locals {
  maybe_issuer = try(data.aws_iam_session_context.current[0].issuer_arn, "")
  key_admins_arns = concat(local.key_admins_arns, [local.maybe_issuer])  # may include ""
}
✅ After (safe: remove blanks; ensure IAM ARN)
hcl
Copy
Edit
data "aws_iam_session_context" "current" { count = 1 }

locals {
  part         = data.aws_partition.current.partition
  issuer_raw   = try(data.aws_iam_session_context.current[0].issuer_arn, "")
  # KMS needs IAM ARNs, not STS. issuer_arn is usually IAM role ARN already; if empty, compact removes it.
  issuer_clean = issuer_raw

  key_admins_with_issuer = compact(concat(local.key_admins_arns, [local.issuer_clean]))
}

module "kms" {
  # ...
  key_administrators = local.key_admins_with_issuer
}
End-to-end example (module call that won’t trigger InvalidArnException)
h
Copy
Edit
data "aws_caller_identity" "current" {}
data "aws_partition" "current" {}
data "aws_iam_session_context" "current" { count = 1 }

variable "kms_key_administrators" { type = list(string) default = [] } # names or ARNs
variable "kms_key_users"          { type = list(string) default = [] } # names or ARNs
variable "repository_role"        { type = string }                    # name or ARN

locals {
  part       = data.aws_partition.current.partition
  account_id = data.aws_caller_identity.current.account_id

  to_role_arn = function(name_or_arn) => (
    can(regex("^arn:", name_or_arn))
      ? name_or_arn
      : "arn:${local.part}:iam::${local.account_id}:role/${name_or_arn}"
  )

  repo_role_arn      = local.to_role_arn(var.repository_role)
  admins_arns_base   = [for a in var.kms_key_administrators : local.to_role_arn(a)]
  users_arns_base    = [for u in var.kms_key_users          : local.to_role_arn(u)]
  issuer_arn         = try(data.aws_iam_session_context.current[0].issuer_arn, "")

  key_admins_arns = compact(concat(local.admins_arns_base, [local.issuer_arn]))
  key_users_arns  = compact(concat(local.users_arns_base, [local.repo_role_arn]))
}

module "kms" {
  source = "terraform-aws-modules/kms/aws"
  # version = "x.y.z"

  create                = true
  description           = "ECR repository encryption key"
  key_usage             = "ENCRYPT_DECRYPT"
  enable_key_rotation   = true
  deletion_window_in_days = 30

  enable_default_policy     = true
  key_owners                = ["arn:${local.part}:iam::${local.account_id}:root"]
  key_administrators        = local.key_admins_arns
  key_users                 = local.key_users_arns
  service_users             = []                # if any, ensure ARNs + compact()
  source_policy_documents   = []
  override_policy_documents = []
  aliases                   = ["alias/ecr/${var.repository_name}"]
}
Checklist you can run through every time

Are all principals IAM ARNs? If any are names, convert with "arn:${partition}:iam::${account_id}:role/${name}".

Did you wrap all lists with compact(...) after concatenation?

Did you remove duplicate attributes in the module block?

If you add issuer_arn, did you run compact(...) afterward so an empty value doesn’t sneak in?

Avoid STS ARNs in KMS key policies (KMS wants IAM ARNs). If you accidentally get an STS assumed-role ARN, convert to the base IAM role ARN.

If you share the exact values you plan to pass (names/ARNs), I can produce the resolved ARNs and a ready-to-apply snippet.









explain this line in details. provide an example how this will look into my code                                                                                                                           variable "kms_key_administrators" { type = list(string) default = [] } # names or ARN


variable "kms_key_administrators" { 
  type    = list(string) 
  default = [] 
} # names or ARN
What it means
variable — This declares an input variable in Terraform.

"kms_key_administrators" — The variable’s name. You’ll use this name when you pass a value in terraform.tfvars or in a -var CLI argument.

type = list(string) — The variable’s type is a list of strings. That means you pass a list ([ ... ]) where each item is a string.

default = [] — If you don’t provide a value, the default will be an empty list ([]).

# names or ARN — This is just a comment telling you the list items can be:

IAM role names (like "MyAdminRole")

Full IAM role ARNs (like "arn:aws:iam::123456789012:role/MyAdminRole")

Why allow names or ARNs?
Because:

In your dev account, you might pass just short role names ("ecr-admin"), and your code will generate the full ARN using the account ID + partition.

In cross-account setups, you often have to pass full ARNs because the role lives in a different account.

Example in your case
Let’s say you want two KMS administrators:

A role in the same account: ECRKeyAdmin

A role in a different AWS account: arn:aws:iam::987654321098:role/CrossAccountKeyAdmin

In variables.tf
You already have:


variable "kms_key_administrators" {
  description = "List of IAM role names or full ARNs that will be KMS key administrators"
  type        = list(string)
  default     = []
}
In terraform.tfvars
You could pass names or ARNs:


kms_key_administrators = [
  "ECRKeyAdmin", # name only, in same account
  "arn:aws:iam::987654321098:role/CrossAccountKeyAdmin" # full ARN for cross-account
]
In your locals (normalize names → ARNs)
h
Copy
Edit
data "aws_caller_identity" "current" {}
data "aws_partition" "current" {}

locals {
  part       = data.aws_partition.current.partition
  account_id = data.aws_caller_identity.current.account_id

  key_admins_arns = [
    for admin in var.kms_key_administrators :
    can(regex("^arn:", admin))
      ? admin
      : "arn:${local.part}:iam::${local.account_id}:role/${admin}"
  ]
}
If you pass the example above, local.key_admins_arns will become:

hcl
Copy
Edit
[
  "arn:aws:iam::123456789012:role/ECRKeyAdmin",              # generated from name
  "arn:aws:iam::987654321098:role/CrossAccountKeyAdmin"      # already an ARN
]
In your KMS module call
hcl
Copy
Edit
module "kms" {
  source = "terraform-aws-modules/kms/aws"
  # ...
  key_administrators = local.key_admins_arns
}
If you want, I can take the exact values you use in your current repo and show you how Terraform will expand them into ARNs before sending them to AWS, so you can see exactly what’s going wrong in your InvalidArnException error.

 to see exactly what Terraform will expand your inputs to, using terraform console. I’ll show:

the normalization code you can drop into your module,

a sample tfvars like yours (mixed names + ARNs), and

the exact console commands + expected output.

1) Normalization code (names → ARNs, remove blanks)
hcl
Copy
Edit
# data sources
data "aws_caller_identity" "current" {}
data "aws_partition" "current" {}

# inputs (yours)
variable "kms_key_administrators" {
  description = "List of IAM role names or full ARNs that will be KMS key administrators"
  type        = list(string)
  default     = []
}

# locals to normalize
locals {
  part       = data.aws_partition.current.partition
  account_id = data.aws_caller_identity.current.account_id

  # Convert any name to IAM role ARN; leave ARNs as-is
  key_admins_arns_raw = [
    for a in var.kms_key_administrators :
    can(regex("^arn:", a)) ? a : "arn:${local.part}:iam::${local.account_id}:role/${a}"
  ]

  # Remove accidental blanks
  key_admins_arns = compact(local.key_admins_arns_raw)
}
Wire it into your KMS module:


module "kms" {
  # ...
  key_administrators = local.key_admins_arns
}
2) Example tfvars (replace with your real values)
hcl
Copy
Edit
# terraform.tfvars (EXAMPLE)
kms_key_administrators = [
  "ECRKeyAdmin",                                            # name in same account
  "arn:aws:iam::987654321098:role/CrossAccountKeyAdmin",    # cross-account ARN
  ""                                                        # oops (blank) → will be removed
]
3) Verify exactly what Terraform computes (terraform console)
From the directory with your code:

bash
Copy
Edit
terraform init    # if not already
terraform console
Now evaluate step-by-step:


# show partition/account
data.aws_partition.current.partition
data.aws_caller_identity.current.account_id

# see raw input
var.kms_key_administrators

# see normalized-without-compact
[
  for a in var.kms_key_administrators :
  can(regex("^arn:", a)) ? a : "arn:${data.aws_partition.current.partition}:iam::${data.aws_caller_identity.current.account_id}:role/${a}"
]

# final list (matches local.key_admins_arns)
compact([
  for a in var.kms_key_administrators :
  can(regex("^arn:", a)) ? a : "arn:${data.aws_partition.current.partition}:iam::${data.aws_caller_identity.current.account_id}:role/${a}"
])
Expected output (example)
Assuming your account is 123456789012 and partition is aws, you should see:

For the non-compact version:

[
  "arn:aws:iam::123456789012:role/ECRKeyAdmin",
  "arn:aws:iam::987654321098:role/CrossAccountKeyAdmin",
  "arn:aws:iam::123456789012:role/"   # <- the blank turned into a bad ARN
]
For the compact (final) version:


[
  "arn:aws:iam::123456789012:role/ECRKeyAdmin",
  "arn:aws:iam::987654321098:role/CrossAccountKeyAdmin"
]
That last step is what prevents InvalidArnException.
