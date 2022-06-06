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
      file: '/templates/renovate-dind.gitlab-ci.yml'
```

If you are using a custom GitLab Kubernetes runner you probably need to downgrade the Docker DinD service because of [containerd/containerd#4837](https://github.com/containerd/containerd/issues/4837)

```yaml
include:
    - project: 'renovate-bot/renovate-runner'
      file: '/templates/renovate-dind.gitlab-ci.yml'

services:
    - docker:19.03.15-dind
```

Alternatively, if you cannot use the gitlab.com hosted or self-hosted privileged runners, include the following template instead.

**Note:** This will use the full renovate image, which isn't capable of respecting any binary contraints.
It will always use the latest tools to update lock files.
So please prefer the DinD version.

```yaml
include:
    - project: 'renovate-bot/renovate-runner'
      file: '/templates/renovate.gitlab-ci.yml'
```

To prevent unexpected changes in your pipeline, you can pin the version of this template and include it in your Renovate updates:

```yaml
include:
    - project: 'renovate-bot/renovate-runner'
      file: '/templates/renovate-dind.gitlab-ci.yml'
      ref: v1.0.0
```

Please check this project's [Releases page](https://gitlab.com/renovate-bot/renovate-runner/-/releases)
to find the latest release tags to reference.

By default our pipeline only runs on schedules.
If you want it to run on other events, see the [GitLab docs for `rules`](https://docs.gitlab.com/ee/ci/yaml/#rules).

Example to run on schedules and pushes:

```yaml
include:
    - project: 'renovate-bot/renovate-runner'
      file: '/templates/renovate-dind.gitlab-ci.yml'

renovate:
    rules:
        - if: '$CI_PIPELINE_SOURCE == "schedule"'
        - if: '$CI_PIPELINE_SOURCE == "push"'
```

## Configure the Schedule

Add a schedule (`CI / CD` > `Schedules`) to run Renovate regularly.

A good practise is to run it hourly. The following runs Renovate on the third minute every hour: `3 * * * *`.

Because the default pipeline only runs on schedules, you need to use the `play` button of schedule to trigger a manual run.

## Other config options

We've changed some renovate defaults for GitLab to better reflect the app's default behavior, so please see [here](./templates/_common.gitlab-ci.yml#L1) for changed options.
For renovate configuration basics checkout the official self-hosting [docs](https://docs.renovatebot.com/self-hosting/#configuration).

For other self-hosted GitLab samples you can check the [Renovate Gitlab Configuration](https://github.com/renovatebot/docker-renovate/blob/HEAD/docs/gitlab.md).

If you are using a self-hosted runner, please checkout the [GitLab docs for Docker DinD configuration](https://docs.gitlab.com/ee/ci/docker/using_docker_build.html#use-the-docker-executor-with-the-docker-image-docker-in-docker).

<b>Note</b>: the default Renovate dind configuration connects to docker over TLS. To enable this e.g. for the GitLab Kubernetes runner add the following:

```toml
[[runners]]
  ...
  [[runners.kubernetes.volumes.empty_dir]]
    name = "docker-certs"
    mount_path = "/certs/client"
    medium = "Memory"
```
