---
stage: Configure
group: Configure
info: To determine the technical writer assigned to the Stage/Group associated with this page, see https://about.gitlab.com/handbook/engineering/ux/technical-writing/#designated-technical-writers
---

# Infrastructure as code with Terraform and GitLab

## Motivation

The Terraform integration features within GitLab enable your GitOps / Infrastructure-as-Code (IaC)
workflows to tie into GitLab's authentication and authorization. These features focus on
lowering the barrier to entry for teams to adopt Terraform, collaborate effectively within
GitLab, and support Terraform best practices.

## GitLab managed Terraform State

> [Introduced](https://gitlab.com/groups/gitlab-org/-/epics/2673) in GitLab 13.0.

[Terraform remote backends](https://www.terraform.io/docs/backends/index.html)
enable you to store the state file in a remote, shared store. GitLab uses the
[Terraform HTTP backend](https://www.terraform.io/docs/backends/types/http.html)
to securely store the state files in local storage (the default) or
[the remote store of your choice](../../administration/terraform_state.md).

The GitLab managed Terraform state backend can store your Terraform state easily and
securely, and spares you from setting up additional remote resources like
Amazon S3 or Google Cloud Storage. Its features include:

- Supporting encryption of the state file both in transit and at rest.
- Locking and unlocking state.
- Remote Terraform plan and apply execution.

To get started with a GitLab-managed Terraform State, there are two different options:

- [Use a local machine](#get-started-using-local-development).
- [Use GitLab CI](#get-started-using-gitlab-ci).

## Permissions for using Terraform

In GitLab version 13.1, [Maintainer access](../permissions.md) was required to use a
GitLab managed Terraform state backend. In GitLab versions 13.2 and greater,
[Maintainer access](../permissions.md) is required to lock, unlock and write to the state
(using `terraform apply`), while [Developer access](../permissions.md) is required to read
the state (using `terraform plan -lock=false`).

## Get started using local development

If you plan to only run `terraform plan` and `terraform apply` commands from your
local machine, this is a simple way to get started:

1. Create your project on your GitLab instance.
1. Navigate to **Settings > General** and note your **Project name**
   and **Project ID**.
1. Define the Terraform backend in your Terraform project to be:

   ```hcl
   terraform {
     backend "http" {
     }
   }
   ```

1. Create a [Personal Access Token](../profile/personal_access_tokens.md) with
   the `api` scope.

1. On your local machine, run `terraform init`, passing in the following options,
   replacing `<YOUR-STATE-NAME>`, `<YOUR-PROJECT-ID>`,  `<YOUR-USERNAME>` and
   `<YOUR-ACCESS-TOKEN>` with the relevant values. This command initializes your
   Terraform state, and stores that state within your GitLab project. This example
   uses `gitlab.com`:

   ```shell
   terraform init \
       -backend-config="address=https://gitlab.com/api/v4/projects/<YOUR-PROJECT-ID>/terraform/state/<YOUR-STATE-NAME>" \
       -backend-config="lock_address=https://gitlab.com/api/v4/projects/<YOUR-PROJECT-ID>/terraform/state/<YOUR-STATE-NAME>/lock" \
       -backend-config="unlock_address=https://gitlab.com/api/v4/projects/<YOUR-PROJECT-ID>/terraform/state/<YOUR-STATE-NAME>/lock" \
       -backend-config="username=<YOUR-USERNAME>" \
       -backend-config="password=<YOUR-ACCESS-TOKEN>" \
       -backend-config="lock_method=POST" \
       -backend-config="unlock_method=DELETE" \
       -backend-config="retry_wait_min=5"
   ```

   NOTE: **Note:**
   The name of your state can contain only uppercase and lowercase letters,
   decimal digits, hyphens and underscores.

You can now run `terraform plan` and `terraform apply` as you normally would.

## Get started using GitLab CI

If you don't want to start with local development, you can also use GitLab CI to
run your `terraform plan` and `terraform apply` commands.

Next, [configure the backend](#configure-the-backend).

## Configure the backend

After executing the `terraform init` command, you must configure the Terraform backend
and the CI YAML file:

1. In your Terraform project, define the [HTTP backend](https://www.terraform.io/docs/backends/types/http.html)
   by adding the following code block in a `.tf` file (such as `backend.tf`) to
   define the remote backend:

   ```hcl
   terraform {
     backend "http" {
     }
   }
   ```

1. In the root directory of your project repository, configure a
   `.gitlab-ci.yaml` file. This example uses a pre-built image which includes a
   `gitlab-terraform` helper. For supported Terraform versions, see the [GitLab
   Terraform Images project](https://gitlab.com/gitlab-org/terraform-images).

   ```yaml
   image: registry.gitlab.com/gitlab-org/terraform-images/stable:latest
   ```

1. In the `.gitlab-ci.yaml` file, define some environment variables to ease
   development. In this example, `TF_ROOT` is the directory where the Terraform
   commands must be executed, `TF_ADDRESS` is the URL to the state on the GitLab
   instance where this pipeline runs, and the final path segment in `TF_ADDRESS`
   is the name of the Terraform state. Projects may have multiple states, and
   this name is arbitrary, so in this example we set it to `example-production`
   which corresponds with the directory we're using as our `TF_ROOT`, and we
   ensure that the `.terraform` directory is cached between jobs in the pipeline
   using a cache key based on the state name (`example-production`):

   ```yaml
   variables:
     TF_ROOT: ${CI_PROJECT_DIR}/environments/example/production
     TF_ADDRESS: ${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/terraform/state/example-production

   cache:
     key: example-production
     paths:
       - ${TF_ROOT}/.terraform
   ```

1. In a `before_script`, change to your `TF_ROOT`:

   ```yaml
   before_script:
     - cd ${TF_ROOT}

   stages:
     - prepare
     - validate
     - build
     - deploy

   init:
     stage: prepare
     script:
       - gitlab-terraform init

   validate:
     stage: validate
     script:
       - gitlab-terraform validate

   plan:
     stage: build
     script:
       - gitlab-terraform plan
       - gitlab-terraform plan-json
     artifacts:
       name: plan
       paths:
         - ${TF_ROOT}/plan.cache
       reports:
         terraform: ${TF_ROOT}/plan.json

   apply:
     stage: deploy
     environment:
       name: production
     script:
       - gitlab-terraform apply
     dependencies:
       - plan
     when: manual
     only:
       - master
   ```

1. Push your project to GitLab, which triggers a CI job pipeline. This pipeline
   runs the `gitlab-terraform init`, `gitlab-terraform validate`, and
   `gitlab-terraform plan` commands.

The output from the above `terraform` commands should be viewable in the job logs.

CAUTION: **Caution:**
Like any other job artifact, Terraform plan data is [viewable by anyone with Guest access](../permissions.md) to the repository.
Neither Terraform nor GitLab encrypts the plan file by default. If your Terraform plan
includes sensitive data such as passwords, access tokens, or certificates, GitLab strongly
recommends encrypting plan output or modifying the project visibility settings.

## Example project

See [this reference project](https://gitlab.com/nicholasklick/gitlab-terraform-aws) using GitLab and Terraform to deploy a basic AWS EC2 within a custom VPC.

## Copy Terraform state between backends

Terraform supports copying the state when the backend is changed or
reconfigured. This can be useful if you need to migrate from another backend to
GitLab managed Terraform state. It's also useful if you need to change the state
name as in the following example:

```shell
PROJECT_ID="<gitlab-project-id>"
TF_USERNAME="<gitlab-username>"
TF_PASSWORD="<gitlab-personal-access-token>"
TF_ADDRESS="https://gitlab.com/api/v4/projects/${PROJECT_ID}/terraform/state/old-state-name"

terraform init \
  -backend-config=address=${TF_ADDRESS} \
  -backend-config=lock_address=${TF_ADDRESS}/lock \
  -backend-config=unlock_address=${TF_ADDRESS}/lock \
  -backend-config=username=${TF_USERNAME} \
  -backend-config=password=${TF_PASSWORD} \
  -backend-config=lock_method=POST \
  -backend-config=unlock_method=DELETE \
  -backend-config=retry_wait_min=5
```

```plaintext
Initializing the backend...

Successfully configured the backend "http"! Terraform will automatically
use this backend unless the backend configuration changes.

Initializing provider plugins...

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

Now that `terraform init` has created a `.terraform/` directory that knows where
the old state is, you can tell it about the new location:

```shell
TF_ADDRESS="https://gitlab.com/api/v4/projects/${PROJECT_ID}/terraform/state/new-state-name"

terraform init \
  -backend-config=address=${TF_ADDRESS} \
  -backend-config=lock_address=${TF_ADDRESS}/lock \
  -backend-config=unlock_address=${TF_ADDRESS}/lock \
  -backend-config=username=${TF_USERNAME} \
  -backend-config=password=${TF_PASSWORD} \
  -backend-config=lock_method=POST \
  -backend-config=unlock_method=DELETE \
  -backend-config=retry_wait_min=5
```

```plaintext
Initializing the backend...
Backend configuration changed!

Terraform has detected that the configuration specified for the backend
has changed. Terraform will now check for existing state in the backends.


Acquiring state lock. This may take a few moments...
Do you want to copy existing state to the new backend?
  Pre-existing state was found while migrating the previous "http" backend to the
  newly configured "http" backend. No existing state was found in the newly
  configured "http" backend. Do you want to copy this state to the new "http"
  backend? Enter "yes" to copy and "no" to start with an empty state.

  Enter a value: yes


Successfully configured the backend "http"! Terraform will automatically
use this backend unless the backend configuration changes.

Initializing provider plugins...

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

If you type `yes`, it will copy your state from the old location to the new
location. You can then go back to running it from within GitLab CI.

## Output Terraform Plan information into a merge request

Using the [GitLab Terraform Report artifact](../../ci/pipelines/job_artifacts.md#artifactsreportsterraform),
you can expose details from `terraform plan` runs directly into a merge request widget,
enabling you to see statistics about the resources that Terraform will create,
modify, or destroy.

Let's explore how to configure a GitLab Terraform Report artifact. You can
either use a pre-built image which includes a `gitlab-terraform` helper as
above, where `gitlab-terraform plan-json` outputs the required artifact, or you
can configure this manually as follows:

1. For simplicity, let's define a few reusable variables to allow us to
   refer to these files multiple times:

   ```yaml
   variables:
     PLAN: plan.cache
     PLAN_JSON: plan.json
   ```

1. Install `jq`, a
   [lightweight and flexible command-line JSON processor](https://stedolan.github.io/jq/).
1. Create an alias for a specific `jq` command that parses out the information we
   want to extract from the `terraform plan` output:

   ```yaml
   before_script:
     - apk --no-cache add jq
     - alias convert_report="jq -r '([.resource_changes[]?.change.actions?]|flatten)|{\"create\":(map(select(.==\"create\"))|length),\"update\":(map(select(.==\"update\"))|length),\"delete\":(map(select(.==\"delete\"))|length)}'"
   ```

   NOTE: **Note:**
   In distributions that use Bash (for example, Ubuntu), `alias` statements are not
   expanded in non-interactive mode. If your pipelines fail with the error
   `convert_report: command not found`, alias expansion can be activated explicitly
   by adding a `shopt` command to your script:

   ```yaml
   before_script:
     - shopt -s expand_aliases
     - alias convert_report="jq -r '([.resource_changes[]?.change.actions?]|flatten)|{\"create\":(map(select(.==\"create\"))|length),\"update\":(map(select(.==\"update\"))|length),\"delete\":(map(select(.==\"delete\"))|length)}'"
   ```

1. Define a `script` that runs `terraform plan` and `terraform show`. These commands
   pipe the output and convert the relevant bits into a store variable `PLAN_JSON`.
   This JSON is used to create a
   [GitLab Terraform Report artifact](../../ci/pipelines/job_artifacts.md#artifactsreportsterraform).
   The Terraform report obtains a Terraform `tfplan.json` file. The collected
   Terraform plan report is uploaded to GitLab as an artifact, and is shown in merge requests.

   ```yaml
   plan:
     stage: build
     script:
       - terraform plan -out=$PLAN
       - terraform show --json $PLAN | convert_report > $PLAN_JSON
     artifacts:
       reports:
         terraform: $PLAN_JSON
   ```

   For a full example using the pre-built image, see [Example `.gitlab-ci.yaml`
   file](#example-gitlab-ciyaml-file).

   For an example displaying multiple reports, see [`.gitlab-ci.yaml` multiple reports file](#multiple-terraform-plan-reports).

1. Running the pipeline displays the widget in the merge request, like this:

   ![Merge Request Terraform widget](img/terraform_plan_widget_v13_2.png)

1. Clicking the **View Full Log** button in the widget takes you directly to the
   plan output present in the pipeline logs:

   ![Terraform plan logs](img/terraform_plan_log_v13_0.png)

### Example `.gitlab-ci.yaml` file

```yaml
default:
  image: registry.gitlab.com/gitlab-org/terraform-images/stable:latest

  cache:
    key: example-production
    paths:
      - ${TF_ROOT}/.terraform

variables:
  TF_ROOT: ${CI_PROJECT_DIR}/environments/example/production
  TF_ADDRESS: ${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/terraform/state/example-production

before_script:
  - cd ${TF_ROOT}

stages:
  - prepare
  - validate
  - build
  - deploy

init:
  stage: prepare
  script:
    - gitlab-terraform init

validate:
  stage: validate
  script:
    - gitlab-terraform validate

plan:
  stage: build
  script:
    - gitlab-terraform plan
    - gitlab-terraform plan-json
  artifacts:
    name: plan
    paths:
      - ${TF_ROOT}/plan.cache
    reports:
      terraform: ${TF_ROOT}/plan.json

apply:
  stage: deploy
  environment:
    name: production
  script:
    - gitlab-terraform apply
  dependencies:
    - plan
  when: manual
  only:
    - master
```

### Multiple Terraform Plan reports

Starting with 13.2, you can display multiple reports on the Merge Request page. The reports will also display the `artifacts: name:`. See example below for a suggested setup.

```yaml
default:
  image:
    name: registry.gitlab.com/gitlab-org/gitlab-build-images:terraform
    entrypoint:
      - '/usr/bin/env'
      - 'PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin'

  cache:
    paths:
      - .terraform

stages:
  - build

.terraform-plan-generation:
  stage: build
  variables:
    PLAN: plan.tfplan
    JSON_PLAN_FILE: tfplan.json
  before_script:
    - cd ${TERRAFORM_DIRECTORY}
    - terraform --version
    - terraform init
    - apk --no-cache add jq
  script:
    - terraform validate
    - terraform plan -out=${PLAN}
    - terraform show --json ${PLAN} | jq -r '([.resource_changes[]?.change.actions?]|flatten)|{"create":(map(select(.=="create"))|length),"update":(map(select(.=="update"))|length),"delete":(map(select(.=="delete"))|length)}' > ${JSON_PLAN_FILE}
  artifacts:
    reports:
      terraform: ${TERRAFORM_DIRECTORY}/${JSON_PLAN_FILE}

review_plan:
  extends: .terraform-plan-generation
  variables:
    TERRAFORM_DIRECTORY: "review/"
  # Review will not include an artifact name

staging_plan:
  extends: .terraform-plan-generation
  variables:
    TERRAFORM_DIRECTORY: "staging/"
  artifacts:
    name: Staging

production_plan:
  extends: .terraform-plan-generation
  variables:
    TERRAFORM_DIRECTORY: "production/"
  artifacts:
    name: Production
```

## Using a GitLab managed Terraform state backend as a remote data source

You can use a GitLab-managed Terraform state as a
[Terraform data source](https://www.terraform.io/docs/providers/terraform/d/remote_state.html).
To use your existing Terraform state backend as a data source, provide the following details
as [Terraform input variables](https://www.terraform.io/docs/configuration/variables.html):

- **address**: The URL of the remote state backend you want to use as a data source.
  For example, `https://gitlab.com/api/v4/projects/<TARGET-PROJECT-ID>/terraform/state/<TARGET-STATE-NAME>`.
- **username**: The username to authenticate with the data source. If you are using a [Personal Access Token](../profile/personal_access_tokens.md) for
  authentication, this is your GitLab username. If you are using GitLab CI, this is `'gitlab-ci-token'`.
- **password**: The password to authenticate with the data source. If you are using a Personal Access Token for
  authentication, this is the token value. If you are using GitLab CI, it is the contents of the `${CI_JOB_TOKEN}` CI variable.

An example setup is shown below:

1. Create a file named `example.auto.tfvars` with the following contents:

   ```plaintext
   example_remote_state_address=https://gitlab.com/api/v4/projects/<TARGET-PROJECT-ID>/terraform/state/<TARGET-STATE-NAME>
   example_username=<GitLab username>
   example_access_token=<GitLab Personal Acceess Token>
   ```

1. Define the data source by adding the following code block in a `.tf` file (such as `data.tf`):

   ```hcl
   data "terraform_remote_state" "example" {
     backend = "http"

     config = {
       address = var.example_remote_state_address
       username = var.example_username
       password = var.example_access_token
     }
   }
   ```

Outputs from the data source can now be referenced within your Terraform resources
using `data.terraform_remote_state.example.outputs.<OUTPUT-NAME>`.

You need at least [developer access](../permissions.md) to the target project
to read the Terraform state.
