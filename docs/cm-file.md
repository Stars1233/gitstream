---
title: gitStream Configuration Overview
description: Learn how to customize gitStream to meet the needs of your organization.
---
# gitStream Configuration Overview

Continuous Merge automation files have a `.cm` extension. In a repository,  gitStream loads and parse the `.cm` directory, which can have multiple automation files, each of which is evaluated independently.

You can edit the `.cm` files and add your own checks and rules. Check out the [Automation examples](/examples).
## Automation rules

There are two types of automation rules: repository level rules and organization level rules.

Repository level rules are set by creating a special `.cm` directory in the repository root. Automation rules are specified in files in this directory, which can have any name but must end with `.cm`.

Organization level rules are defined by creating a special repository named `cm` in the organization or group. In this repository, you can add CM automation files, which will apply to all the repositories that gitStream app is connected.

**When organization level rules are defines, repository level automation shall take precedence and override organization automation when having the same identifier.**

An automation identifier is a composition of the CM file name and the automation name. For example when `safe_changes` is defined in `gitstream.cm` then the automation identifier shall be `gitstream/safe_changes`

!!! tip

    You can select (include or exclude) certain repositories per automation file using the [`triggers.include.repository`](./execution-model.md#triggers-section) and [`triggers.exclude.repository`](./execution-model.md#triggers-section)

### Repository automation rules

Repository automation rules are set by creating a special `.cm` directory in your repository root. Automation rules are specified in files in this directory, these files can have any name but must end with `.cm`. By default, you start with a single automation file `.cm/gitstream.cm`.

Every file is parsed independently, and the parsing results are combined and executed.

Specifically:

1. Automation rules are allowed to have same name in different `.cm` files
2. The `config` section is defined per `.cm` file (except `config.admin`)
3. Any accessory expression defined in each file scope, therefore cannot be reused in another file (but it can be duplicated)

When configured correctly, your repository directory structure should look like that (for GitHub):

```title="Repsository automation rules"
.
├─ .cm/
│  └─ *.cm
├─ .github/
│  └─ workflows/
│     └─ gitstream.yml
```

!!! note

    The `.cm/gitstream.cm` is special, as it allows repository-level configuration such as `config.admin`.

### Organization automation rules

Organization automation rules are defined by creating a special repository `cm` in your organization or group. In this repository, you can add CM automation files, which will apply to all the repositories that gitStream app is connected.

When configured correctly, the `cm` repository directory structure should look like that (for GitHub):

```title="Repsository automation rules"
.
├─ *.cm
├─ .github/
│  └─ workflows/
│     └─ gitstream.yml
```

For each PR the following automation rules are applied:

1. Repository level rules
2. Organization-level rules, unless with the same identifier as a repository-level automation

When organization level rules are defined, then the CI/CD will be executed on the `cm` repository on behalf of the PR repository.

### Setting up Global Automation rules

By utilizing the following techniques, you can effectively combine and manage global and repository rules to customize the behavior of your automations to fit the specific requirements of your repositories:

1. Global rules are defined in the configuration management `cm` repository and are applied to all repositories that are connected to gitStream.
2. To exclude or run only on specific repositories from a global rule, you can use `triggers.include.repository` and `triggers.exclude.repository` in the `cm` file and add a list of the unwanted or wanted repositories respectfully.
3. To override a global rule for specific automation in a repository, you can duplicate the rule (both the file and automation name) and place it in the desired repository. The locally defined rule will then take precedence over the global rule for that specific repository.

### Controlling Organization-Level Rules per Repository

The `triggers` section in organization-level rules allows you to precisely control which repositories will run specific automations. This is particularly useful for implementing standardized rules across most repositories while exempting specific ones, or for creating specialized automations that only apply to certain repository types.

#### Repository Inclusion and Exclusion

You can use `triggers.include.repository` and `triggers.exclude.repository` to define which repositories should or should not run particular automations:

```yaml+jinja
# In your organization's 'cm' repository
manifest:
  version: 1.0

# This automation will run on all repositories EXCEPT those containing 'legacy' in their name
triggers:
  exclude:
    repository:
      - r/legacy/

automations:
  enforce_pr_title_format:
    if:
      - {{ not (pr.title | includes(term="fix:|feat:|docs:|style:|refactor:|test:|chore:")) }}
    run:
      - action: add-comment@v1
        args:
          comment: |
            Please format your PR title according to our [Conventional Commits](https://www.conventionalcommits.org/) standard.
            Example: `feat: implement new login flow`
```

#### Creating Repository-Type Specific Rules

You can also create separate CM files for different types of repositories:

```yaml+jinja
# backend-rules.cm in your organization's 'cm' repository
manifest:
  version: 1.0

# This file's automations will ONLY run on repositories with 'api' or 'service' in their name
triggers:
  include:
    repository:
      - r/api|service/

automations:
  require_api_docs:
    if:
      - {{ files | match(regex=r/\/api\/.*\.js|ts$/) | some }}
    run:
      - action: request-changes@v1
        args:
          comment: |
            API changes detected. Please ensure you've updated the corresponding documentation.
```

This approach allows you to create specialized rule sets for different repository types (frontend, backend, infrastructure, etc.) while maintaining them all in a centralized location.


## The .cm automation file
### Schema

The following sections are used in `.cm` file to describe the desired automations:

- [`manifest`](#manifest)
- [`config`](#config)
- [`triggers`](#triggers)
- [`automations`](#automations)

#### `manifest`

The first section in a `gitstream.cm` file is the `manifest`.

```yaml+jinja
manifest:
  version: 1.0
```

The only field required is `version`.

| Key         | Required | Type    | Description                              |
| ----------- | ---------|---------|----------------------------------------- |
| `manifest`         | Y        | Map     | The manifest section root                |
| `manifest.version` | Y        | String  | Specify the `.cm` spec version: 0.1, 1.0 |

The manifest version field is used to parse the `.cm` file, in the future if breaking changes are introduced to the parser then older automation will be still supported.

#### `config`

The `config` section in the `.cm` file is optional and specifies settings that affect gitStream's operation within a given context.

| Key                      | Type             | Default | Scope          | Description                                                                         |
| ------------------------ | ---------------- | ------- | -------------- | ----------------------------------------------------------------------------------- |
| `config`                 | Map              | -       | per `.cm` file | Root configuration section, applies to the automations defined in the current file. |
| `config.admin.users`     | [String]         | `[]`    | `gitstream.cm` | List of admin users, identified by Git provider usernames.                          |
| `config.git_history_since` | String         | -       | per `.cm` file | Limit git history analysis to commits after the specified date (YYYY-MM-DD).       |
| `config.ignore_files`    | [String]         | `[]`    | per `.cm` file | Files to exclude from consideration.                                                |
| `config.user_mapping`    | [String: String] | `[]`    | per `.cm` file | Map Git user details to provider account names.                                     |

##### `config.admin.users`

When specified in `gitstream.cm`, the `config.admin.users` allows adding admin rights. When a PR changes the `*.cm` files only, if the user is listed in `config.admin.users`, the PR will be approved by gitStream. Furthermore, if an admin is defined, they will automatically be requested to review changes in PRs that change rules files and will also be set as a required reviewer, meaning gitStream will block the merge until the admin approves the PR.

For example, setting `popeye` as admin:

```yaml title="example"
config:
  admin:
    users: ['popeye']
```

This configuration is valid only when used in `.cm/gitstream.cm`, when defined in other `.cm` files this configuration is ignored.

When you add a user to `config.admin.users` in your organization's `cm` repository, they are granted administrative privileges to CM changes across **every repository** in the organization. gitStream evaluates CM rules in the individual repository **and** your organization's `cm` repository to determine admin users.

When you add a user to `config.admin.users` in your repository's `.cm/gitstream.cm` file, they are granted administrative privileges to CM changes within that specific repository. However, if an organization-level admin is already defined in your organization's `cm` repository, then the organization-level admin approval is required for that change to take effect.

##### `config.ignore_files`

The `config.ignore_files` supports glob pattern matching that contains a list of files to ignore.

Common usage, since some files such as lock files are intentionally not a required part of a review, they would not want to them to be counted in the estimated review time. In such cases, you can add `config.ignore_files` to the relevant CM file, for example:

```yaml title="example"
config:
  ignore_files:
    - 'yarn.lock'
    - 'package-lock.json'
    - 'openapi.json'
    - 'ui/src/**/*Model.d.ts'
```

##### `config.user_mapping`

Accepts list of key value strings.

For example, when using `rankByGitBlame` or `explainRankByGitBlame` Git users are mapped to their matching Git provider accounts based on the Git details. The automatic mapping can sometimes result with the wrong account or fail to find a proper mapping, in these cases you can configure the `config.user_mapping`. This allows you to map confusing Git user into their specific accounts and dump some irrelevant accounts:

```yaml title="example"
config:
  user_mapping:
    - 'Popeye Man <popeye@invalid.com>': 'popeye-the-salyor-man' # (1)
    - 'Popeye Man <popeye2@invalid.com>': 'popeye-the-salyor-man' # (2)
    - 'olive <olive@invalid.com>': null # (3)
```

1.  Map Git user signature to the correct Git provider user name (e.g. GitHub)
2.  Map Git user signature to the correct Git provider user name (e.g. GitHub)
2.  Mapping to `null` removes this Git user from the suggested results

When using `rankByGitBlame` to assign reviewers automatically with `add-reviewers@v1` then mapping users to `null` is a way to prevent the automatic mapping in certain cases, like in your example contributors that are not longer part of the team.

On the other hand, when using `explainRankByGitBlame` with `add-comment@v1` it still shows these users details in the PR comment suggestion as this info might be valuable by itself.

```yaml+jinja
- action: add-reviewers@v1
  args: # (1)
    reviewers: {{ repo | rankByGitBlame(gt=25) }}

- action: add-comment@v1
  args: # (2)
    comment: |
      {{ repo | explainRankByGitBlame(gt=25) }}
```

1.  `rankByGitBlame` will drop `null` users
2.  `explainRankByGitBlame` will NOT drop `null` users

##### `config.git_history_since`

The `config.git_history_since` configuration limits how far back gitStream looks in git history when analyzing code expertise and git blame results. This affects filters and actions that rely on git history analysis, including:

- `codeExperts` filter
- `rankByGitBlame` filter  
- `explainRankByGitBlame` filter
- `explain-code-experts` action

When `git_history_since` is configured, gitStream will only consider commits made on or after the specified date when determining code experts and calculating git blame rankings. This is particularly useful for:

- **Team transitions**: When a project changes hands between teams, you can set the date to when the new team took over to ensure only current team members are suggested as reviewers
- **Performance optimization**: Large repositories with extensive history can benefit from limiting the analysis scope
- **Relevant expertise**: Focus on recent contributions rather than legacy code changes

The value should be specified as a date in `YYYY-MM-DD` format.

```yaml title="example"
config:
  git_history_since: '2025-01-01'  # Only consider commits from January 1, 2025 onwards
```

**Example use case**: If your team took over a project on January 1, 2025, and you want to ensure that only current team members are suggested as code experts:

```yaml title=".cm/gitstream.cm"
config:
  git_history_since: '2025-01-01'

automations:
  assign_code_experts:
    if:
      - {{ files | codeExperts(gt=10) | length > 0 }}
    run:
      - action: add-reviewers@v1
        args:
          reviewers: {{ files | codeExperts(gt=10) }}
```

With this configuration, the `codeExperts` filter will only analyze git blame data from commits made on or after January 1, 2025, effectively filtering out previous team members from reviewer suggestions.

#### `triggers`

The `triggers` is section specifies when automations are executed, supporting `include` and `exclude` lists for branch and repository patterns at the file level. The `on` keyword can also be used within individual automations to define specific events that trigger those automations.

```yaml+jinja
triggers:
  on:
    - pr_created
    - commit
  exclude:
    branch:
      - hotfix
```

Read more here: [triggers](/execution-model#explicit-triggers).

#### `automations`

The `automations` section defines the automations and their conditions.

```yaml+jinja
automations:
  mark_small_pr:
    if:
      - {{ checks.size.is.xsmall }}
    run:
      - action: add-label@v1
        args:
          label: xsmall
```

Each automation includes its name, and few fields: `if` and `run`.

| Key        | Required  | Type    | Description                                     |
|------------|-----------|---------|------------------------------------------------ |
| `automations`  | Y        | Map     | The automations section root     |
| `automations.NAME`     | Y | Map | User defined name of the automation, can be any string       |
| `automations.NAME.if`  | Y | Map | List of conditions with AND relationship |
| `automations.NAME.run` | Y | Map | Actions to run if all conditions are met, invoked one by one |

The `if` field includes the list of conditions. The conditions are checked when a pull request
is opened or changed, if all the conditions pass, the automation is executed.

The `run` field includes the automation to execute. It includes the following fields:

| Key         | Required | Type    | Description                                     |
| ----------- | ---------|---------|------------------------------------------------ |
| `action`    | Y        | String  | The action pointer                              |
| `engine`    | N        | String  | The action engine, default is `gitstream`       |
| `args`      | N        | List    | The action inputs list                          |

For `gitstream` engine, the action is specified by: `name@version`

gitStream supported actions, see [actions](/automation-actions).

### Reusing checks

You can define an accessory section, e.g. `checks`, that defines common conditions, and reuse.

```yaml+jinja
size:
  is:
    small: {{ branch.diff.size < 20 }}
    medium: {{ branch.diff.size >= 20 and branch.diff.size < 100 }}
    large: {{ branch.diff.size >= 100 }}

automations:
  approve_small:
    if:
      - {{ size.is.small }}
    run:
      - action: approve@v1
  mark_small_medium:
    if:
      # Check that the PR is either small or medium size
      - {{ size.is.small or size.is.medium }}
      # AND its less than 5 minutes review (estimated)
      - {{ branch | estimatedReviewTime <= 5 }}
    run:
      - action: add-label@v1
        args:
          label: 'good-size'

```
