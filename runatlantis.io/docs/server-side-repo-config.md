# Server Side Repo Config
A Server-Side Repo Config file is used to control per-repo behaviour
and what users can do in repo-level `atlantis.yaml` files.

[[toc]]

## Do I Need A Server-Side Repo Config File?
You do not need a server-side repo config file unless you want to customize
some aspect of Atlantis on a per-repo basis.

Read through the [use-cases](#use-cases) to determine if you need it.

## Enabling Server Side Repo Config
To use server side repo config create a config file, ex. `repos.yaml`, and pass it to
the `atlantis server` command via the `--repo-config` flag, ex. `--repo-config=path/to/repos.yaml`.

If you don't wish to write a config file to disk, you can use the
`--repo-config-json` flag or `ATLANTIS_REPO_CONFIG_JSON` environment variable
to specify your config as JSON. See [--repo-config-json](server-configuration.html#repo-config-json)
for an example.

## Example Server Side Repo
```yaml
# repos lists the config for specific repos.
repos:
  # id can either be an exact repo ID or a regex.
  # If using a regex, it must start and end with a slash.
  # Repo ID's are of the form {VCS hostname}/{org}/{repo name}, ex.
  # github.com/runatlantis/atlantis.
- id: /.*/

  # apply_requirements sets the Apply Requirements for all repos that match.
  apply_requirements: [approved, mergeable]

  # workflow sets the workflow for all repos that match.
  # This workflow must be defined in the workflows section.
  workflow: custom

  # allowed_overrides specifies which keys can be overridden by this repo in
  # its atlantis.yaml file.
  allowed_overrides: [apply_requirements, workflow]

  # allow_custom_workflows defines whether this repo can define its own
  # workflows. If false (default), the repo can only use server-side defined
  # workflows.
  allow_custom_workflows: true

  # id can also be an exact match.
- id: github.com/myorg/specific-repo

# workflows lists server-side custom workflows
workflows:
  custom:
    plan:
      steps:
      - run: my-custom-command arg1 arg2
      - init
      - plan:
          extra_args: ["-lock", "false"]
      - run: my-custom-command arg1 arg2
    apply:
      steps:
      - run: echo hi
      - apply
 ```

## Use Cases
Here are some of the reasons you might want to use a repo config.

### Requiring PR Is Approved Before Apply
If you want to require that all (or specific) repos must have pull requests
approved before Atlantis will allow running `apply`, use the `apply_requirements` key.

For all repos:
```yaml
# repos.yaml
repos:
- id: /.*/
  apply_requirements: [approved]
```

For a specific repo:
```yaml
# repos.yaml
repos:
- id: github.com/myorg/myrepo
  apply_requirements: [approved]
```

See [Apply Requirements](apply-requirements.html) for more details.

### Requiring PR Is "Mergeable" Before Apply
If you want to require that all (or specific) repos must have pull requests
in a mergeable state before Atlantis will allow running `apply`, use the `apply_requirements` key.

For all repos:
```yaml
# repos.yaml
repos:
- id: /.*/
  apply_requirements: [mergeable]
```

For a specific repo:
```yaml
# repos.yaml
repos:
- id: github.com/myorg/myrepo
  apply_requirements: [mergeable]
```

See [Apply Requirements](apply-requirements.html) for more details.

### Repos Can Set Their Own Apply Requirements
If you want all (or specific) repos to be able to override the default apply requirements, use
the `allowed_overrides` key.

To allow all repos to override the default:
```yaml
# repos.yaml
repos:
- id: /.*/
  # The default will be approved.
  apply_requirements: [approved]

  # But all repos can set their own using atlantis.yaml
  allowed_overrides: [apply_requirements]
```
To allow only a specific repo to override the default:
```yaml
# repos.yaml
repos:
# Set a default for all repos.
- id: /.*/
  apply_requirements: [approved]

# Allow a specific repo to override.
- id: github.com/myorg/myrepo
  allowed_overrides: [apply_requirements]
```

Then each allowed repo can have an `atlantis.yaml` file that
sets `apply_requirements` to an empty array (disabling the requirement).
```yaml
# atlantis.yaml in the repo root
version: 3
projects:
- dir: .
  apply_requirements: []
```

### Change The Default Atlantis Workflow
If you want to change the default commands that Atlantis runs during `plan` and `apply`
phases, you can create a new `workflow`.

If you want to use that workflow by default for all repos, use the workflow
key `default`:
```yaml
# repos.yaml
# NOTE: the repos key is not required.
workflows:
  # It's important that this is "default".
  default:
    plan:
      steps:
      - init
      - run: my custom plan command
    apply:
      steps:
      - run: my custom apply command
```

See [Custom Workflows](custom-workflows.html) for more details on writing
custom workflows.

### Allow Repos To Choose A Server-Side Workflow
If you want repos to be able to choose their own workflows that are defined
in the server-side repo config, you need to create the workflows
server-side and then allow each repo to override the `workflow` key:

```yaml
# repos.yaml
# Allow repos to override the workflow key.
repos:
- id: /.*/
  allowed_overrides: [workflow]

# Define your custom workflows.
workflows:
  custom1:
    plan:
      steps:
      - init
      - run: my custom plan command
    apply:
      steps:
      - run: my custom apply command

  custom2:
    plan:
      steps:
      - run: another custom command
    apply:
      steps:
      - run: another custom command
```

Then each allowed repo can choose one of the workflows in their `atlantis.yaml`
files:

```yaml
# atlantis.yaml
version: 3
projects:
- dir: .
  workflow: custom1 # could also be custom2 OR default
```

:::tip NOTE
There is always a workflow named `default` that corresponds to Atlantis' default workflow
unless you've created your own server-side workflow with that key (overriding it).
:::

See [Custom Workflows](custom-workflows.html) for more details on writing
custom workflows.

### Allow Repos To Define Their Own Workflows
If you want repos to be able to define their own workflows you need to
allow them to override the `workflow` key and set `allow_custom_workflows` to `true`.

::: danger
If repos can define their own workflows, then anyone that can create a pull
request to that repo can essentially run arbitrary code on your Atlantis server.
:::

```yaml
# repos.yaml
repos:
- id: /.*/

  # With just allowed_overrides: [workflow], repos can only
  # choose workflows defined server-side.
  allowed_overrides: [workflow]

  # By setting allow_custom_workflows to true, we allow repos to also
  # define their own workflows.
  allow_custom_workflows: true
```

Then each allowed repo can define and use a custom workflow in their `atlantis.yaml` files:
```yaml
# atlantis.yaml
version: 3
projects:
- dir: .
  workflow: custom1
workflows:
  custom1:
    plan:
      steps:
      - init
      - run: my custom plan command
    apply:
      steps:
      - run: my custom apply command
```

See [Custom Workflows](custom-workflows.html) for more details on writing
custom workflows.

## Reference

### Top-Level Keys
| Key       | Type                                                    | Default   | Required | Description                                                                           |
|-----------|---------------------------------------------------------|-----------|----------|---------------------------------------------------------------------------------------|
| repos     | array[[Repo](#repo)]                                    | see below | no       | List of repos to apply settings to.                                                   |
| workflows | map[string: [Workflow](custom-workflows.html#workflow)] | see below | no       | Map from workflow name to workflow. Workflows override the default Atlantis commands. |


::: tip A Note On Defaults
#### `repos`
`repos` always contains a first element with the Atlantis default config:
```yaml
repos:
- id: /.*/
  apply_requirements: []
  workflow: default
  allowed_overrides: []
  allow_custom_workflows: false
```

#### `workflows`
`workflows` always contains the Atlantis default workflow under the key `default`:
```yaml
workflows:
  default:
    plan:
      steps: [init, plan]
    apply:
      steps: [apply]
```
This gets merged with whatever config you write.
If you set a workflow with the key `default`, it will override this.
:::

### Repo
| Key                    | Type     | Default | Required | Description                                                                                                                                                                                                                                                                                              |
|------------------------|----------|---------|----------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| id                     | string   | none    | yes      | Value can be a regular expression when specified as /&lt;regex&gt;/ or an exact string match. Repo IDs are of the form `{vcs hostname}/{org}/{name}`, ex. `github.com/owner/repo`. Hostname is specified without scheme or port. For Bitbucket Server, {org} is the **name** of the project, not the key. |
| workflow               | string   | none    | no       | A custom workflow.                                                                                                                                                                                                                                                                                       |
| apply_requirements     | []string | none    | no       | Requirements that must be satisfied before `atlantis apply` can be run. Currently the only supported requirements are `approved` and `mergeable`. See [Apply Requirements](apply-requirements.html) for more details.                                                                                    |
| allowed_overrides      | []string | none    | no       | A list of restricted keys that `atlantis.yaml` files can override. The only supported keys are `apply_requirements` and `workflow`                                                                                                                                                                       |
| allow_custom_workflows | bool     | false   | no       | Whether or not to allow [Custom Workflows](custom-workflows.html).                                                                                                                                                                       |


:::tip Notes
* If multiple repos match, the last match will apply.
* If a key isn't defined, it won't override a key that matched from above.
  For example, given a repo ID `github.com/owner/repo` and a config:
  ```yaml
  repos:
  - id: /.*/
    allow_custom_workflows: true
    apply_requirements: [approved]
  - id: github.com/owner/repo
    apply_requirements: []
  ```

  The final config will look like:
  ```yaml
  apply_requirements: []
  workflow: default
  allowed_overrides: []
  allow_custom_workflows: true
  ```
  Where
  * `apply_requirements` is set from the `id: github.com/owner/repo` config because
    it overrides the previous matching config from `id: /.*/`.
  * `workflow` is set from the default config that always
    exists.
  * `allowed_overrides` is set from the default config that always
    exists.
  * `allow_custom_workflows` is set from the `id: /.*/` config and isn't unset
    by the `id: github.com/owner/repo` config because it didn't define that key.
:::
