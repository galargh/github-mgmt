# GitHub Management via Terraform: Template

This repository is meant to serve as a template for creating new repositories responsible for managing GitHub configuration as code with Terraform. It provides an opinionated way to manage GitHub configuration without following the Terraform usage guidelines to the letter. It was designed with managing multiple, medium to large sized GitHub organisations in mind and that is exactly what it is/is going to be optimised for.

## Key features

- 2-way sync between GitHub Management and the actual GitHub configuration (including bootstrapping)
- PR-based configuration change review process which guarantees the reviewed plan is the one being applied
- control over what resources and what properties are managed by GitHub Management

## How does it work?

GitHub Management allows management of GitHub configuration as code. It uses Terraform and GitHub Actions to achieve this.

The JSON configuration files for a specific organisation are stored in [github/$ORGANIZATION_NAME](github/$ORGANIZATION_NAME) directory. GitHub Management lets you manage multiple organizations from a single repository. It uses separate terraform workspaces per each organisation. The local workspaces are called like the organisations themselves while the remote Terraform Cloud workspaces get `org_` prefix added.

The configuration files are named after the [GitHub Provider resources](https://registry.terraform.io/providers/integrations/github/latest/docs) they configure but are stripped of the `github_` prefix.

Each configuration file contains a single, top-level *JSON* object. The keys in that object are resource identifiers - the required argument values of that resource type. The values - objects describing other arguments, attributes and the id of that resource type.

For example, [github_repository](https://registry.terraform.io/providers/integrations/github/latest/docs/resources/repository#argument-reference) resource has one required argument - `name` - so the keys inside the object in [repository.json](github/$ORGANIZATION_NAME/repository.json) would be the names of the repositories owned by the organisation. The values in that object would be objects describing remaining [arguments](https://registry.terraform.io/providers/integrations/github/latest/docs/resources/repository#argument-reference) and [attributes](https://registry.terraform.io/providers/integrations/github/latest/docs/resources/repository#attributes-reference).

Another example would be [github_branch_protection](https://registry.terraform.io/providers/integrations/github/latest/docs/resources/branch_protection#argument-reference) which has two required arguments - `repository_id` and `pattern`. In such case, the keys would be nested under each other. The keys in the top-level object in [branch_protection.json](github/$ORGANIZATION_NAME/branch_protection.json) would be the IDs of the repositories owned by the organisation and their values would be objects with patterns of branch protection rules for that repository as keys. Where possible, the IDs in key values are replaced with more human friendly values. That's why, the actual top-level key values in [branch_protection.json](github/$ORGANIZATION_NAME/branch_protection.json) would be repository names, not repository IDs.

Whether resources of a specific type are managed via GitHub Management or not is controlled by the existence of the corresponding configuration files. If such a file exists, GitHub Management manages all the arguments and attributes of that resource type except for the ones specified in the `ignore_changes` lists in [terraform/resources.tf](terraform/resources.tf).

GitHub Management is capable of both applying the changes made to the JSON configuration files to the actual GitHub configuraiton state and of translating the current GitHub configuration state into the JSON configuration files.

The workflow for introducing changes to GitHub via JSON configuration files is as follows:
1. Modify the JSON configuration file.
1. Create a PR and wait for the GitHub Action workflow triggered on PRs to comment on it with a terraform plan.
1. Review the plan.
1. Merge the PR and wait for the GitHub Action workflow triggered on pushes to the default branch to apply it.

Neither creating the terraform plan nor applying it refreshes the underlying terraform state i.e. going through this workflow does **NOT** ask GitHub if the actual GitHub configuration state has changed. This makes the workflow fast and rate limit friendly because the number of requests to GitHub is minimised. This can result in the plan failing to be applied, e.g. if the underlying resource has been deleted. This assumes that JSON configuration should be the main source of truth for GitHub configuration state. The plans that are created during the PR GitHub Action workflow are applied exactly as-is after the merge.

The workflow for synchronising the current GitHub configuration state with JSON configuration files is as follows:
1. Run the `Sync` GitHub Action workflow and wait for the PR to be created.
1. If a PR was created, close and reopen it (because PRs created with the default `GITHUB_TOKEN` do not trigger workflows by default) and wait for the GitHub Action workflow triggered on PRs to comment on it with a terraform plan.
1. Ensure that the plan introduces no changes.
1. Merge the PR.

Running the `Sync` GitHub Action workflows refreshes the underlying terraform state. It also automatically imports all the resources that were created outside GitHub Management into the state and removes any that were deleted. After the `Sync` flow, all the other open PRs should have their GitHub Action workflows rerun because merging them without it would result in the application of their plans to fail due to the plans being created against a different state.

## Supported resources

| Resource | JSON | Key(s) | Dependencies | Description |
| --- | --- | --- | --- | --- |
| [github_membership](https://registry.terraform.io/providers/integrations/github/latest/docs/resources/membership) | `membership.json` | `username` | n/a | add/remove users from your organization |
| [github_repository](https://registry.terraform.io/providers/integrations/github/latest/docs/resources/repository) | `repository.json` | `repository.name` | n/a | create and manage repositories within your GitHub organization |
| [github_repository_collaborator](https://registry.terraform.io/providers/integrations/github/latest/docs/resources/repository_collaborator) | `repository_collaborator.json` | `repository.name`: `username` | `github_repository` | add/remove collaborators from repositories in your organization |
| [github_branch_protection](https://registry.terraform.io/providers/integrations/github/latest/docs/resources/branch_protection) | `branch_protection.json` | `repository.name`: `pattern` | `github_repository` | configure branch protection for repositories in your organization |
| [github_team](https://registry.terraform.io/providers/integrations/github/latest/docs/resources/team) | `team.json` | `team.name` | n/a | add/remove teams from your organization |
| [github_team_repository](https://registry.terraform.io/providers/integrations/github/latest/docs/resources/team_repository) | `team_repository.json` | `team.name`: `repository.name` | `github_team` | manage relationships between teams and repositories in your GitHub organization |
| [github_team_membership](https://registry.terraform.io/providers/integrations/github/latest/docs/resources/team_membership) | `team_membership.json` | `team.name`: `username` | `github_team` | add/remove users from teams in your organization |

## How to...

### ...get started?

*NOTE*: The following TODO list is complete - it contains all the steps you should complete to get GitHub Management up. You might be able to skip some of them if you completed them before.

#### Terraform Cloud

- [ ] [Create a Terraform Cloud workspace](https://www.terraform.io/cloud-docs/workspaces/creating) for `API-driven workflow` called `org_$GITHUB_ORGANIZATION_NAME` (\*replace `$GITHUB_ORGANIZATION_NAME` with the GitHub organisation name) - *this is where Terraform state for the organisation will be stored*
- [ ] Change the [execution mode](https://www.terraform.io/cloud-docs/workspaces/settings#execution-mode) of the Terraform Cloud workspace to `Local` under `Settings > General Settings` - *remote execution has not been tested; it is also assumed that local execution will make requests to GitHub faster since they will be made from GitHub Action machines*

#### GitHub App

- [ ] [Create a GitHub App](https://docs.github.com/en/developers/apps/building-github-apps/creating-a-github-app) in the GitHub organisation with the following permissions - *this is how terraform will be able to authenticate with GitHub*:
    - `Repository permissions`
        - `Administration`: `Read & Write`
        - `Contents`: `Read & Write`
        - `Metadata`: `Read-only`
    - `Organization permissions`
        - `Members`: `Read & Write`
- [ ] [Install the GitHub App](https://docs.github.com/en/developers/apps/managing-github-apps/installing-github-apps) in the GitHub organisation for `All repositories`

#### GitHub Organisation Secrets

- [ ] [Create encrypted secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-an-organization) for the GitHub organisation and allow the repository to access them (\*replace `$GITHUB_ORGANIZATION_NAME` with the GitHub organisation name) - *these secrets are read by the GitHub Action workflows*
    - [ ] `TF_GITHUB_APP_ID_$GITHUB_ORGANIZATION_NAME`: Go to `https://github.com/organizations/$GITHUB_ORGANIZATION_NAME/settings/apps/$GITHUB_APP_NAME` and copy the `App ID`
    - [ ] `TF_GITHUB_APP_INSTALLATION_ID_$GITHUB_ORGANIZATION_NAME`: Go to `https://github.com/organizations/$GITHUB_ORGANIZATION_NAME/settings/installations`, click `Configure` next to the `$GITHUB_APP_NAME` and copy the numeric suffix from the URL
    - [ ] `TF_GITHUB_APP_PEM_FILE_$GITHUB_ORGANIZATION_NAME`: Go to `https://github.com/organizations/$GITHUB_ORGANIZATION_NAME/settings/apps/$GITHUB_APP_NAME`, click `Generate a private key` and copy the contents of the downloaded PEM file
    - [ ] `TF_API_TOKEN`: Go to https://app.terraform.io/app/settings/tokens, click `Create an API Token` and copy the value that is eventually displayed

#### GitHub Management Repository

- [ ] [Create a repository](https://docs.github.com/en/repositories/creating-and-managing-repositories/creating-a-repository-from-a-template) from the template - *this is the place for GitHub Management to live in*
- [ ] [Clone the repository](https://docs.github.com/en/repositories/creating-and-managing-repositories/cloning-a-repository)
- [ ] Replace placeholder strings in the clone - *the repository needs to be customised for the specific organisation it is supposed to manage*
    - [ ] Rename the `$GITHUB_ORGANIZATION_NAME` directory in `github` to the name of the GitHub organisation
    - [ ] Replace `$GITHUB_ORGANIZATION_NAME` with the name of the GitHub organisation in `github/$GITHUB_ORGANIZATION_NAME/branch_protection.json`
    - [ ] Replace `$GITHUB_MGMT_REPOSITORY_NAME` with the name of the repository in `github/$GITHUB_ORGANIZATION_NAME/branch_protection.json` and `github/$GITHUB_ORGANIZATION_NAME/repository.json`
    - [ ] Replace `$GITHUB_MGMT_REPOSITORY_DEFAULT_BRANCH` with the name of the default branch in the repository in `github/$GITHUB_ORGANIZATION_NAME/branch_protection.json` and `.github/workflows/push.yml`

#### GitHub Management Sync Flow

- [ ] Follow [How to synchronize GitHub Management with GitHub?](#synchronize-github-management-with-github) while using the `branch` with your changes as a target to import all the resources you want to manage for the organisation

### ...add an organisation to be managed by GitHub Management?

- [ ] Follow [How to get started with Terraform Cloud?](#terraform-cloud) to create a remote workspace
- [ ] Follow [How to get started with GitHub App?](#github-app) to create a GitHub App for the organisation
- [ ] Follow [How to get started with GitHub Organization Secrets?](#github-organisation-secrets) to set up secrets that GitHub Management is going to use
- [ ] Create a new directory called like the organisation under [github](github) directory which is going to store the configuration files
- [ ] Follow [How to add a resource type to be managed by GitHub Management?](#add-a-resource-type-to-be-managed-by-github-management) to add some resources to be managed by GitHub Management
- [ ] Follow [How to synchronize GitHub Management with GitHub?](#synchronize-github-management-with-github) while using the `branch` with your changes as a target to import all the resources you want to manage for the organisation

### ...add a resource type to be managed by GitHub Management?

- [ ] Create a new JSON file with `{}` as content for one of the [supported resources](#supported-resources) under `github/$ORGANIZATION_NAME` directory
- [ ] Follow [How to synchronize GitHub Management with GitHub?](#synchronize-github-management-with-github) while using the `branch` with your changes as a target to import all the resources you want to manage for the organisation

### ...add a resource argument/attribute to be managed by GitHub Management?

*NOTE*: You cannot set the values of attributes via GitHub Management but sometimes it is useful to have them available in the configuration files. For example, it might be a good idea to have `github_team.id` unignored if you want to manage `github_team.parent_team_id` via GitHub Management so that the users can quickly check each team's id without leaving the JSON configuration file.

- [ ] Comment out the argument/attribute you want to start managing using GitHub Management in [terraform/resources.tf](terraform/resources.tf)
- [ ] Follow [How to synchronize GitHub Management with GitHub?](#synchronize-github-management-with-github) while using the `branch` with your changes as a target to import all the resources you want to manage for the organisation

### ...add a resource?

*NOTE*: You do not have to specify all the arguments/attributes when creating a new resource. If you don't, defaults as defined by the [GitHub Provider](https://registry.terraform.io/providers/integrations/github/latest/docs) will be used. The next `Sync` will fill out the remaining arguments/attributes in the JSON configuration file.

*NOTE*: When creating a new resource, you can specify all the arguments that the resource supports even if changes to them are ignored. If you do specify arguments to which changes are ignored, their values are going to be applied during creation but a future `Sync` will remove them from configuration JSON.

- [ ] Add a new JSON object `{}` under unique key in the JSON configuration file for one of the [supported resource](#supported-resources)
- [ ] Follow [How to apply GitHub Management changes to GitHub?](#apply-github-management-changes-to-github) to create your newly added resource

### ...modify a resource?

- [ ] Change the value of an argument/attribute in the JSON configuration file for one of the [supported resource](#supported-resources)
- [ ] Follow [How to apply GitHub Management changes to GitHub?](#apply-github-management-changes-to-github) to create your newly added resource

### ...apply GitHub Management changes to GitHub?

- [ ] [Create a pull request](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/creating-a-pull-request) from the branch to the default branch
- [ ] Merge the pull request once the `Plan ($GITHUB_ORGANIZATION_NAME)` check passes and you verify the plan posted as a comment
- [ ] Confirm that the `Push` GitHub Action workflow run applied the plan by inspecting the output

### ...synchronize GitHub Management with GitHub?

*NOTE*: Remember that the `Sync` operation modifes terraform state. Even if you run it from a branch, it modifies the global state that is shared with other branches. There is only one terraform state per organisation.

*NOTE*: If you run the `Sync` from an unprotected branch, then the workflow will commit changes to it directly.

*Note*: `Sync` is also going to sort the keys in all the objects lexicographically.

- [ ] Run `Sync` GitHub Action workflow from your desired `branch` - *this will import all the resources from the actual GitHub configuration state into GitHub Management*
- [ ] Close and reopen the pull request that the `Sync` GitHub Action workflow run created or updated
- [ ] Merge the pull request once the `Plan ($GITHUB_ORGANIZATION_NAME)` check passes and you verify the plan posted as a comment - *the plan should not contain any changes*
