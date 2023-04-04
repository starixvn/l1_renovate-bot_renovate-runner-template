# Renovate Runner

The intention of this project is to provide a pipeline which is easy to set up and reflects the current app settings as close as possible.

You will need to:

1. Create a new project to host the runner
2. Configure credentials using CI variables
3. Create a new `main` pipeline that includes this project's template
4. Set up a schedule to run the pipeline regularly

## Create a new Runner project

We recommend you use a new and dedicated private project to host the Renovate runner, however a public project with private CI logs should still be safe.
Currently one advantage of public projects is that CI minutes are not restricted however the same restrictions as private projects will soon apply.

## Configure CI/CD variables

You need to add a GitLab [Personal Access Token](https://docs.gitlab.com/ee/user/profile/personal_access_tokens.html#creating-a-personal-access-token) (scopes: `read_user`, `api` and `write_repository`) as `RENOVATE_TOKEN` to CI/CD variables.
You can also use a GitLab [Group Access Token](https://docs.gitlab.com/ee/user/group/settings/group_access_tokens.html).
Checkout Renovate platform [docs](https://docs.renovatebot.com/modules/platform/gitlab/) and #53 for more information.

It is also recommended to configure a [GitHub.com Personal Access Token](https://docs.github.com/en/free-pro-team@latest/github/authenticating-to-github/creating-a-personal-access-token) (minimum scopes) as `GITHUB_COM_TOKEN` so that your bot can make authenticated requests to github.com for Changelog retrieval as well as for any dependency that uses GitHub tags.
Without such a token, github.com's API will rate limit requests and make such lookups unreliable.

Finally, you need to decide how your bot should decide which projects to run against.
By default renovate won't find any repo, you need to choose one of the following options for `RENOVATE_EXTRA_FLAGS`.

If you wish for your bot to run against any project which the `RENOVATE_TOKEN` PAT has access to, but which already have a `renovate.json` or similar config file, then add this variable: `RENOVATE_EXTRA_FLAGS=`: `--autodiscover=true`.
This will mean no new projects will be onboarded.

However, we recommend you apply an `autodiscoverFilter` value like the following so that the bot does not run on any stranger's project it gets invited to: `RENOVATE_EXTRA_FLAGS`: `--autodiscover=true --autodiscover-filter=group1/*`.
Checkout renovate [docs](https://docs.renovatebot.com/gitlab-bot-security/) for more information about gitlab security.

If you wish for your bot to run against _every_ project which the `RENOVATE_TOKEN` PAT has access to, and onboard any projects which don't yet have a config, then add this variable: `RENOVATE_EXTRA_FLAGS`: `--autodiscover=true --onboarding=true --autodiscover-filter=group1/*`.

If you wish to manually specify which projects that your bot runs again, then add this variable with a space-delimited set of project names: `RENOVATE_EXTRA_FLAGS`: `group1/repo5 user3/repo1`.

## Create a GitLab CI file

Create a `.gitlab-ci.yml` file in the repository like the following:

```yaml
include:
    - project: 'renovate-bot/renovate-runner'
      file: '/templates/renovate.gitlab-ci.yml'
```

**Note:** Docker DinD templates are deprecated and is now using the default version with slim image.

Alternatively, if you can use the full renovate image.
It will have the latest tools to update lock files preinstalled and is much bigger.
So please prefer the default version.

```yaml
include:
    - project: 'renovate-bot/renovate-runner'
      file: '/templates/renovate.gitlab-ci.yml'

image: ${CI_RENOVATE_IMAGE_FULL}
```

To prevent unexpected changes in your pipeline, you can pin the version of this template and include it in your Renovate updates:

```yaml
include:
    - project: 'renovate-bot/renovate-runner'
      file: '/templates/renovate.gitlab-ci.yml'
      ref: v12.0.0
```

Please check this project's [Releases page](https://gitlab.com/renovate-bot/renovate-runner/-/releases)
to find the latest release tags to reference.

By default our pipeline only runs on schedules.
If you want it to run on other events, see the [GitLab docs for `rules`](https://docs.gitlab.com/ee/ci/yaml/#rules).

Example to run on schedules and pushes:

```yaml
include:
    - project: 'renovate-bot/renovate-runner'
      file: '/templates/renovate.gitlab-ci.yml'

renovate:
    rules:
        - if: '$CI_PIPELINE_SOURCE == "schedule"'
        - if: '$CI_PIPELINE_SOURCE == "push"'
```

You can now use remote includes to use these templates on self-hosted gitlab instances.
The folling samples uses the `v12.0.0` tag.
You can also use `main` branch but a tag is preferred.
Refer to GitLab [include](https://docs.gitlab.com/ee/ci/yaml/includes.html) samples for more information.

```yaml
include:
    - remote: https://gitlab.com/renovate-bot/renovate-runner/-/raw/v12.0.0/templates/renovate.gitlab-ci.yml
```

## Configure the Schedule

Add a schedule (`CI / CD` > `Schedules`) to run Renovate regularly.

A good practise is to run it hourly. The following runs Renovate on the third minute every hour: `3 * * * *`.

Because the default pipeline only runs on schedules, you need to use the `play` button of schedule to trigger a manual run.

## Other config options

We've changed some renovate defaults for GitLab to better reflect the app's default behavior, so please see [here](./templates/renovate.gitlab-ci.yml#L1) for changed options.
Onboarding and autodiscover is disabled by default for security reasons.
For renovate configuration basics checkout the official self-hosting [docs](https://docs.renovatebot.com/self-hosted-configuration/).

For other self-hosted GitLab samples you can check the [Renovate Gitlab Configuration](https://github.com/renovatebot/docker-renovate/blob/HEAD/docs/gitlab.md).
