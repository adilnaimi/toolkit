---
description: Github Actions(CI/CD) tricks that official documents didn't mention. This post including hints, tips, snippet, cheatsheet, troubleshooting, notes, how-to.
---

# Github Action

Github Actions(CI/CD) tricks that official documents didn't mention. This post including hints, tips, snippet, cheatsheet, troubleshooting, notes, how-to.

## Starter
```yaml
on:
  push:
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Run tests
        run: |
          echo hi
```

## On trigger event
```yaml

on:
  push:
    branches:
      - "master"
      - "**"
    tags:
      - "**"
  pull_request:
    branches:
      - master
  repository_dispatch:
    types: [rammus_post]
```

!!! important
    `pull_request.branches` is base on **ref**, not **head_ref**

## Environments and variables
### Pass variables
```bash
echo ::set-output name=message::$output_message
echo ::set-env name=action_state::yellow
```
### Use variables
```yaml
GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
PR_COMMENT_URL: ${{ github.event.pull_request.comments_url }}
PR_COMMENT: ${{ steps.message.outputs.message }}
PAYLOAD_ACTOR: ${{ github.event.client_payload.actor }}
```
### Use GITHUB_ACTOR as git commit author and using github avator
https://help.github.com/en/github/setting-up-and-managing-your-github-user-account/setting-your-commit-email-address
```yaml
git config --global user.name "${GITHUB_ACTOR}"
git config --global user.email "${GITHUB_ACTOR}@users.noreply.github.com"
remote_repo="https://x-access-token:${GITHUB_TOKEN}@github.com/${GITHUB_REPOSITORY}.git"
git remote add origin "${remote_repo}"
```

#### Get commit author information
This is only for ==push== event
```yaml
AUTHOR_NAME: ${{ github.event.head_commit.author.name }}
AUTHOR_EMAIL: ${{ github.event.head_commit.author.email }}
```

## if condition
```yaml
if: contains(github.ref, 'tags')
if: steps.git-diff.outputs.is-diff
if: steps.set-env.outputs.message == 'hello'
if: github.ref != 'refs/heads/master'

```

### pull_request
#### when specific event type
```yaml
if: github.event_name == 'pull_request' && github.event.action == 'unassigned'
```
#### when branch name is
```yaml
if: github.event_name == 'pull_request' && contains(github.head_ref, 'my-feature-branch')
```
#### when merged
```yaml
on: 
  pull_request:
    types: [closed]
jobs:
  merged:    
    if: github.event.pull_request.merged == true
```

## Setting
```yaml
ACTIONS_RUNNER_DEBUG: true
```

## Build an action

```yaml
# action.yml
name: 'Hello World'
description: 'Greet someone and record the time'
inputs:
  who-to-greet:  # id of input
    description: 'Who to greet'
    required: true
    default: 'World'
outputs:
  time: # id of output
    description: 'The time we greeted you'
runs:
  using: 'docker'
  image: 'Dockerfile'
  args:
    - ${{ inputs.who-to-greet }}
```

### Get input as environemnt in Docker
```yaml
# action.yaml
inputs:
  who-to-greet:
```
```yaml
# workflow.yaml
      - uses: ./actions/my-action
        with:
          who-to-greet: rammus
```

```bash
# entrypoint.sh
echo INPUTS_WHO_TO_GREET
```

### ENTRYPOINT need to be abolute path

!!! warning
    When `uses: ./actions/my-action`
    Workflow will mount workspace `--workdir /github/workspace`

```bash
COPY ./app.py /
ENTRYPOINT python /app.py
```

## Awesome Actions

### Customize action type with http post method
```yaml
on: 
  repository_dispatch:
    types: [rammus_post]
jobs:
  rammus_job:
    env:
      ACTOR: ${{ github.event.client_payload.actor }}
    if: github.event.action == 'rammus_post'
```

And you can send a post request like:

```bash
INPUT_GITHUB_TOKEN=
INPUT_COMMENT_URL="https://api.github.com/repos/<owner>/<repo>/dispatches"

curl -H "Authorization: token $INPUT_GITHUB_TOKEN" \
    -H "Accept: application/vnd.github.everest-preview+json" \
    -d '{"event_type":"rammus_post","client_payload":{"new_version_image":"echo-server:96b2166", "new_version_sha":"96b2166","actor":"Rammus Xu"}}' \
    -XPOST $INPUT_COMMENT_URL
```

### Docker login 
```yaml
    - name: Docker Login - docker.pkg.github.com
      uses: swaglive/actions/docker/login@944b742
      with:
        password: ${{ secrets.GHR_PASSWORD }}
        username: ${{ secrets.GHR_USERNAME }}
        url: docker.pkg.github.com

    - name: Docker Login - docker.pkg.github.com
      if: contains(github.ref, 'tags')
      env: 
        DOCKER_PASSWORD: ${{ secrets.GITHUB_TOKEN }}
      run: echo $DOCKER_PASSWORD | docker login -u $GITHUB_ACTOR --password-stdin docker.pkg.github.com
    
```

### Docker login and copy config to default container's workspace

!!! note
    github runner will mount `/home/runner/work/_temp/_github_home":"/github/home` when we use a docker action.
    That means we can't use the credential directly in next steps. Only if you use a docker contain step. Since `id: generate-mirror-list` need docker credentials and run on default container, credentials should be copied to default container's home.

    And a docker action must run as root. Therefore, it needs to be `sudo` in a default container.

```yaml
    - name: Docker Login - docker.pkg.github.com
      uses: swaglive/actions/docker/login@944b742
      with:
        password: ${{ secrets.GHR_PASSWORD }}
        username: ${{ secrets.GHR_USERNAME }}
        url: docker.pkg.github.com

    - name: Do something in default container's workspace
      run: |
        sudo cp -R /home/runner/work/_temp/_github_home/.docker ~
        sudo chown -R $(whoami) ~/.docker
        docker pull docker.pkg.github.com/swaglive/dockerfiles/kubectl:1.17
```

### Cache node_modules
```yaml
      - uses: actions/cache@v1
        with:
          path: ./node_modules
          key: ${{ runner.os }}-yarn-${{ hashFiles('**/yarn.lock') }}
          restore-keys: |
            ${{ runner.os }}-yarn-
```

### Fetch private submodule
PAT = [private access token](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line)
```yaml
    - name: Fix submodules
      run: echo -e '[url "https://github.com/"]\n  insteadOf = "git@github.com:"' >> ~/.gitconfig
    - name: Checkout
      uses: actions/checkout@v1
      with:
        fetch-depth: 0
        submodules: true
        token: ${{ secrets.PAT }}
```

### Create pull request
```yaml
    - name: Create Pull Request
      if: steps.create-branch.outputs.branch_name
      uses: actions/github-script@0.3.0
      with:
        github-token: ${{ secrets.GITHUB_TOKEN }}
        script: |
          github.pulls.create({
            owner: 'swaglive',
            repo: 'action-demo',
            title: '[Action] ${{ steps.create-branch.outputs.branch_name }}',
            head: '${{ steps.create-branch.outputs.branch_name }}',
            base: 'master'
          })
    - name: Create Pull Request
      if: steps.create-branch.outputs.branch_name
      uses: actions/github-script@0.3.0
      with:
        github-token: ${{ secrets.GITHUB_TOKEN }}
        script: |
          github.pulls.create({
            ...context.repo,
            title: '[Action] ${{ steps.create-branch.outputs.branch_name }}',
            body: 'Create by `${{ github.event.client_payload.actor }}`',
            head: '${{ steps.create-branch.outputs.branch_name }}',
            base: 'master'
          })
```

### Close issue when title contains specific string
```yaml
name: issue-opened

on:
  issues:
    types: [opened]

jobs:
  debug:
    runs-on: ubuntu-latest
    if: contains(github.event.issue.title, 'Update APK')
    steps:
      - name: Close issue
        uses: actions/github-script@0.8.0
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            github.issues.update({
              ...context.repo,
              issue_number: context.issue.number,
              state: 'closed'
            })

            github.issues.createComment({
              ...context.repo,
              issue_number: context.issue.number,
              body: 'Close this!'
            });
```

### Create a comment in pull request
```yaml
    - name: Notify Results in Pull Request
      if: steps.generate-mirror-list.outputs.images
      uses: actions/github-script@0.3.0
      with:
        github-token: ${{ secrets.GITHUB_TOKEN }}
        images: ${{ steps.generate-mirror-list.outputs.images }}
        script: |
          var body = "## I mirrored something for you"
          process.env.INPUT_IMAGES.split(' ').forEach(function(image){
            let [source, target] = image.split('@')
            body = `${body}\r\n- \`${source}\` -> \`${target}\``
          })

          github.issues.createComment({
            ...context.repo,
            issue_number: context.payload.number,
            body: body,
          })
```

### Deploy mkdocs to gh-pages
```yaml
name: Publish
on:
  push:
    branches:
      - master

jobs:
  deploy:
    name: Deploy docs
    runs-on: ubuntu-latest
    steps:
      - name: Checkout master
        uses: actions/checkout@v1

      - name: Deploy docs
        uses: mhausenblas/mkdocs-deploy-gh-pages@1.11
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Get environments in action
```yaml
jobs:
  show-env:
    runs-on: ubuntu-latest
    steps:
    - run: cat $GITHUB_EVENT_PATH
    - run: echo ${{ github.event_name }}
    - run: echo ${{ github.event.action }}
    - run: env
```

### Get other step output as enviroment in github-script
```yaml
jobs:
  debug:
    runs-on: ubuntu-latest
    steps:
      - id: update
        run: echo ::set-output name=message::okok
      - name: js
        uses: actions/github-script@0.8.0
        env:
          MESSAGE: ${{ steps.update.outputs.message }}
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            var message = process.env.MESSAGE
            if (message !== undefined){
              message == 'in'
            } else {
              message == 'else'
            }
            console.log(message)
```

## Othes
### iOS App Build
Compare:

- bitrise: 2 vCPU + 4 GB RAM
- Github Action: 2 vCPU + 7 GB RAM

### Build status
```yaml tab="Clickable"
[![Publish](https://github.com/RammusXu/toolkit/workflows/Publish/badge.svg)](https://github.com/RammusXu/toolkit)
```

```yaml tab="Unclickable"
![Publish](https://github.com/RammusXu/toolkit/workflows/Publish/badge.svg)
```

- Clickable: [![Publish](https://github.com/RammusXu/toolkit/workflows/Publish/badge.svg)](https://github.com/RammusXu/toolkit)
- Unclickable: ![Publish](https://github.com/RammusXu/toolkit/workflows/Publish/badge.svg)

## Env Sample
### 2020-03-11 Use a self build action
```bash
/usr/bin/docker run --name e87b528978e939d04c5cabae17f516da08bb2a_79b912 --label e87b52 --workdir /github/workspace --rm -e INPUT_ARGS -e INPUT_MY_VAR -e INPUT_LOWER_VAR -e INPUT_WHO-TO-GREET -e INPUT_NAME -e HOME -e GITHUB_REF -e GITHUB_SHA -e GITHUB_REPOSITORY -e GITHUB_RUN_ID -e GITHUB_RUN_NUMBER -e GITHUB_ACTOR -e GITHUB_WORKFLOW -e GITHUB_HEAD_REF -e GITHUB_BASE_REF -e GITHUB_EVENT_NAME -e GITHUB_WORKSPACE -e GITHUB_ACTION -e GITHUB_EVENT_PATH -e RUNNER_OS -e RUNNER_TOOL_CACHE -e RUNNER_TEMP -e RUNNER_WORKSPACE -e ACTIONS_RUNTIME_URL -e ACTIONS_RUNTIME_TOKEN -e ACTIONS_CACHE_URL -e GITHUB_ACTIONS=true -v "/var/run/docker.sock":"/var/run/docker.sock" -v "/home/runner/work/_temp/_github_home":"/github/home" -v "/home/runner/work/_temp/_github_workflow":"/github/workflow" -v "/home/runner/work/action-demo/action-demo":"/github/workspace" e87b52:8978e939d04c5cabae17f516da08bb2a env
```
### 2020-02-25 on a push branch
```bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=7e68d28ee47e
HOME=/github/home
GITHUB_REF=refs/heads/show-env
GITHUB_REPOSITORY=rammusxu/action-demo
GITHUB_RUN_ID=42801200
GITHUB_RUN_NUMBER=153
GITHUB_ACTOR=RammusXu
RUNNER_TEMP=/home/runner/work/_temp
INPUT_ARGS=env
ACTIONS_RUNTIME_URL=https://pipelines.actions.githubusercontent.com/PMQIhg54lq7cRaMuQESXLNDyCqKLwtTaQOTGOPlUgbwmwZae5C/
RUNNER_WORKSPACE=/home/runner/work/action-demo
GITHUB_ACTIONS=true
GITHUB_EVENT_NAME=push
GITHUB_WORKFLOW=RammusPush
GITHUB_HEAD_REF=
GITHUB_BASE_REF=
GITHUB_ACTION=alpine
GITHUB_EVENT_PATH=/github/workflow/event.json
RUNNER_OS=Linux
RUNNER_TOOL_CACHE=/opt/hostedtoolcache
GITHUB_SHA=7f4bedfc1262dbb55052aa18ecf9d21388c50526
ACTIONS_RUNTIME_TOKEN=***
GITHUB_WORKSPACE=/github/workspace
```

```json
{
  "after": "7f4bedfc1262dbb55052aa18ecf9d21388c50526",
  "base_ref": null,
  "before": "0000000000000000000000000000000000000000",
  "commits": [
    {
      "author": {
        "email": "comte_ken@hotmail.com",
        "name": "Rammus Xu",
        "username": "RammusXu"
      },
      "committer": {
        "email": "comte_ken@hotmail.com",
        "name": "Rammus Xu",
        "username": "RammusXu"
      },
      "distinct": true,
      "id": "7f4bedfc1262dbb55052aa18ecf9d21388c50526",
      "message": "parallel --help",
      "timestamp": "2020-02-21T13:36:19+08:00",
      "tree_id": "f4f9fd21306ee5fb29fab80d373ef55b1852c082",
      "url": "https://github.com/swaglive/action-demo/commit/7f4bedfc1262dbb55052aa18ecf9d21388c50526"
    }
  ],
  "compare": "https://github.com/swaglive/action-demo/commit/7f4bedfc1262",
  "created": true,
  "deleted": false,
  "forced": false,
  "head_commit": {
    "author": {
      "email": "comte_ken@hotmail.com",
      "name": "Rammus Xu",
      "username": "RammusXu"
    },
    "committer": {
      "email": "comte_ken@hotmail.com",
      "name": "Rammus Xu",
      "username": "RammusXu"
    },
    "distinct": true,
    "id": "7f4bedfc1262dbb55052aa18ecf9d21388c50526",
    "message": "parallel --help",
    "timestamp": "2020-02-21T13:36:19+08:00",
    "tree_id": "f4f9fd21306ee5fb29fab80d373ef55b1852c082",
    "url": "https://github.com/swaglive/action-demo/commit/7f4bedfc1262dbb55052aa18ecf9d21388c50526"
  },
  "organization": {
    "avatar_url": "https://avatars3.githubusercontent.com/u/36289044?v=4",
    "description": "The SWAG Life",
    "events_url": "https://api.github.com/orgs/swaglive/events",
    "hooks_url": "https://api.github.com/orgs/swaglive/hooks",
    "id": 36289044,
    "issues_url": "https://api.github.com/orgs/swaglive/issues",
    "login": "swaglive",
    "members_url": "https://api.github.com/orgs/swaglive/members{/member}",
    "node_id": "MDEyOk9yZ2FuaXphdGlvbjM2Mjg5MDQ0",
    "public_members_url": "https://api.github.com/orgs/swaglive/public_members{/member}",
    "repos_url": "https://api.github.com/orgs/swaglive/repos",
    "url": "https://api.github.com/orgs/swaglive"
  },
  "pusher": {
    "email": "comte_ken@hotmail.com",
    "name": "RammusXu"
  },
  "ref": "refs/heads/show-parallel",
  "repository": {
    "archive_url": "https://api.github.com/repos/swaglive/action-demo/{archive_format}{/ref}",
    "archived": false,
    "assignees_url": "https://api.github.com/repos/swaglive/action-demo/assignees{/user}",
    "blobs_url": "https://api.github.com/repos/swaglive/action-demo/git/blobs{/sha}",
    "branches_url": "https://api.github.com/repos/swaglive/action-demo/branches{/branch}",
    "clone_url": "https://github.com/swaglive/action-demo.git",
    "collaborators_url": "https://api.github.com/repos/swaglive/action-demo/collaborators{/collaborator}",
    "comments_url": "https://api.github.com/repos/swaglive/action-demo/comments{/number}",
    "commits_url": "https://api.github.com/repos/swaglive/action-demo/commits{/sha}",
    "compare_url": "https://api.github.com/repos/swaglive/action-demo/compare/{base}...{head}",
    "contents_url": "https://api.github.com/repos/swaglive/action-demo/contents/{+path}",
    "contributors_url": "https://api.github.com/repos/swaglive/action-demo/contributors",
    "created_at": 1564547431,
    "default_branch": "master",
    "deployments_url": "https://api.github.com/repos/swaglive/action-demo/deployments",
    "description": null,
    "disabled": false,
    "downloads_url": "https://api.github.com/repos/swaglive/action-demo/downloads",
    "events_url": "https://api.github.com/repos/swaglive/action-demo/events",
    "fork": false,
    "forks": 0,
    "forks_count": 0,
    "forks_url": "https://api.github.com/repos/swaglive/action-demo/forks",
    "full_name": "swaglive/action-demo",
    "git_commits_url": "https://api.github.com/repos/swaglive/action-demo/git/commits{/sha}",
    "git_refs_url": "https://api.github.com/repos/swaglive/action-demo/git/refs{/sha}",
    "git_tags_url": "https://api.github.com/repos/swaglive/action-demo/git/tags{/sha}",
    "git_url": "git://github.com/swaglive/action-demo.git",
    "has_downloads": true,
    "has_issues": true,
    "has_pages": false,
    "has_projects": true,
    "has_wiki": true,
    "homepage": null,
    "hooks_url": "https://api.github.com/repos/swaglive/action-demo/hooks",
    "html_url": "https://github.com/swaglive/action-demo",
    "id": 199778774,
    "issue_comment_url": "https://api.github.com/repos/swaglive/action-demo/issues/comments{/number}",
    "issue_events_url": "https://api.github.com/repos/swaglive/action-demo/issues/events{/number}",
    "issues_url": "https://api.github.com/repos/swaglive/action-demo/issues{/number}",
    "keys_url": "https://api.github.com/repos/swaglive/action-demo/keys{/key_id}",
    "labels_url": "https://api.github.com/repos/swaglive/action-demo/labels{/name}",
    "language": "Dockerfile",
    "languages_url": "https://api.github.com/repos/swaglive/action-demo/languages",
    "license": null,
    "master_branch": "master",
    "merges_url": "https://api.github.com/repos/swaglive/action-demo/merges",
    "milestones_url": "https://api.github.com/repos/swaglive/action-demo/milestones{/number}",
    "mirror_url": null,
    "name": "action-demo",
    "node_id": "MDEwOlJlcG9zaXRvcnkxOTk3Nzg3NzQ=",
    "notifications_url": "https://api.github.com/repos/swaglive/action-demo/notifications{?since,all,participating}",
    "open_issues": 4,
    "open_issues_count": 4,
    "organization": "swaglive",
    "owner": {
      "avatar_url": "https://avatars3.githubusercontent.com/u/36289044?v=4",
      "email": "engineering@swag.live",
      "events_url": "https://api.github.com/users/swaglive/events{/privacy}",
      "followers_url": "https://api.github.com/users/swaglive/followers",
      "following_url": "https://api.github.com/users/swaglive/following{/other_user}",
      "gists_url": "https://api.github.com/users/swaglive/gists{/gist_id}",
      "gravatar_id": "",
      "html_url": "https://github.com/swaglive",
      "id": 36289044,
      "login": "swaglive",
      "name": "swaglive",
      "node_id": "MDEyOk9yZ2FuaXphdGlvbjM2Mjg5MDQ0",
      "organizations_url": "https://api.github.com/users/swaglive/orgs",
      "received_events_url": "https://api.github.com/users/swaglive/received_events",
      "repos_url": "https://api.github.com/users/swaglive/repos",
      "site_admin": false,
      "starred_url": "https://api.github.com/users/swaglive/starred{/owner}{/repo}",
      "subscriptions_url": "https://api.github.com/users/swaglive/subscriptions",
      "type": "Organization",
      "url": "https://api.github.com/users/swaglive"
    },
    "private": false,
    "pulls_url": "https://api.github.com/repos/swaglive/action-demo/pulls{/number}",
    "pushed_at": 1582263389,
    "releases_url": "https://api.github.com/repos/swaglive/action-demo/releases{/id}",
    "size": 184,
    "ssh_url": "git@github.com:swaglive/action-demo.git",
    "stargazers": 0,
    "stargazers_count": 0,
    "stargazers_url": "https://api.github.com/repos/swaglive/action-demo/stargazers",
    "statuses_url": "https://api.github.com/repos/swaglive/action-demo/statuses/{sha}",
    "subscribers_url": "https://api.github.com/repos/swaglive/action-demo/subscribers",
    "subscription_url": "https://api.github.com/repos/swaglive/action-demo/subscription",
    "svn_url": "https://github.com/swaglive/action-demo",
    "tags_url": "https://api.github.com/repos/swaglive/action-demo/tags",
    "teams_url": "https://api.github.com/repos/swaglive/action-demo/teams",
    "trees_url": "https://api.github.com/repos/swaglive/action-demo/git/trees{/sha}",
    "updated_at": "2020-02-15T04:12:28Z",
    "url": "https://github.com/swaglive/action-demo",
    "watchers": 0,
    "watchers_count": 0
  },
  "sender": {
    "avatar_url": "https://avatars1.githubusercontent.com/u/4367069?v=4",
    "events_url": "https://api.github.com/users/RammusXu/events{/privacy}",
    "followers_url": "https://api.github.com/users/RammusXu/followers",
    "following_url": "https://api.github.com/users/RammusXu/following{/other_user}",
    "gists_url": "https://api.github.com/users/RammusXu/gists{/gist_id}",
    "gravatar_id": "",
    "html_url": "https://github.com/RammusXu",
    "id": 4367069,
    "login": "RammusXu",
    "node_id": "MDQ6VXNlcjQzNjcwNjk=",
    "organizations_url": "https://api.github.com/users/RammusXu/orgs",
    "received_events_url": "https://api.github.com/users/RammusXu/received_events",
    "repos_url": "https://api.github.com/users/RammusXu/repos",
    "site_admin": false,
    "starred_url": "https://api.github.com/users/RammusXu/starred{/owner}{/repo}",
    "subscriptions_url": "https://api.github.com/users/RammusXu/subscriptions",
    "type": "User",
    "url": "https://api.github.com/users/RammusXu"
  }
}
```


## Reference
- https://help.github.com/en/actions/reference
- https://github.com/actions/toolkit/tree/master/packages/github
- https://github.com/actions/github-script
- https://octokit.github.io/rest.js/#usage
