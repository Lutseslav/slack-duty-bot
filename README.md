# Slack duty bot

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Go Report Card](https://goreportcard.com/badge/github.com/iqoption/slack-duty-bot)](https://goreportcard.com/report/github.com/iqoption/slack-duty-bot)
[![Build Status](https://travis-ci.org/iqoption/slack-duty-bot.svg?branch=master)](https://travis-ci.org/iqoption/slack-duty-bot)

## Usage

1. Create new custom integration `Bots` (e.g https://{team}.slack.com/apps/manage/custom-integrations)
2. Add bot to channels you want to listen
3. Build for your environment or download [tarball with binary](https://github.com/iqoption/slack-duty-bot/releases) for your OS and arch
4. Prepare config.yaml with duties list
5. Run with the required parameters

```bash
SDB_SLACK_TOKEN=your-token-here ./slack-duty-bot \
    --slack.keyword keyword-1 \
    --slack.keyword keyword-2 \
    --slack.group.id your-group-id \
    --slack.group.name your-group-name
```

You also can run in application in docker:

```bash
docker run \
    --name slack-duty-bot \
    --restart on-failure \
    -v $(pwd)/config:/etc/slack-duty-bot \
    -e SDB_SLACK_TOKEN=your-token-here \
    -d \
    iqoption/slack-duty-bot:1.1.0 \
    --config.path=/etc/slack-duty-bot \
    --slack.keyword keyword-1 \
    --slack.keyword keyword-2
    --slack.group.id your-group-id \
    --slack.group.name your-group-name
```

### Build package

** Golang:1.11 is required **

Build

```bash
env GOOS=linux GOARCH=amd64 GO111MODULE=on go build -o slack-duty-bot -v
```

Build via makefile

```bash
make GOOS=linux GOARCH=amd64
```

Build in docker

```bash
docker run \
    --rm \
    -v $(pwd):/project \
    -w /project \
    -e GOOS=linux \
    -e GOARCH=amd64 \
    -e GO111MODULE=on \
    golang:1.11 go build -o slack-duty-bot -v
```

### Configuration flags, environment variables

Environment variables are prefixed with `SDB_` and **MUST** be uppercase with `_` delimiter

Available variables:

* `SDB_SLACK_TOKEN`
* `SDB_SLACK_GROUP_ID`
* `SDB_SLACK_GROUP_NAME`
* `SDB_SLACK_THREADS`
* `SDB_LOGGER_LEVEL`

Every environment variable can be overwritten by startup flags

Available flags:

* `--slack.token` - Slack API client token (**Required**)
* `--slack.keyword` - Case insensitive keywords slice to search in message text, can be set multiple times (default: []) (**Required**)
* `--config.path` - Path to config.yaml file (default: . and `$HOME/.slack-duty-bot`)
* `--slack.group.name` - Slack user group name, to mention in channel if duty list is empty
* `--slack.group.id` - Slack user group ID, to mention in channel if duty list is empty
* `--slack.threads` - Use threads as reply target or push message direct to channel (default: `true`)
* `--logger.level` - Log level (default: `info`)

You can get IDS from api or just use [testing page](https://api.slack.com/methods/usergroups.list/test)

### Configuration file
There are three ways of configuration for `slack-duty-bot`
#### Slack group
```
slack:
  group:
    id: some-group-id
    name: some-group-name
```
In this way `slack-duty-bot` will always call entire Slack group in case matching keyword in message.
#### Duties list
By default `duties` is an empty slice (or just skipped in config)
```yaml
duties: []
```
In this case you **MUST** set `slack.group` configuration

If you prefer to set different duties for different days, you **MUST** set `duties` as slice, containing **7** indexes of slices:
```yaml
duties:
  - [username.one, username.two] # Sunday
  - [username.one] # Monday
  - [username.two] # Tuesday
  - [username.one] # Wednesday
  - [username.two] # Thursday
  - [username.one] # Friday
  - [username.one, username.two] # Saturday
```
#### Combined
If you prefer to set custom duties for some days, and call full group for another, just pass `duties` slice and `slack.group` configuration together:
```yaml
slack:
  group:
    id: some-group-id
    name: some-group-name
duties:
  - [] # Sunday
  - [username.one] # Monday
  - [username.two] # Tuesday
  - [username.one] # Wednesday
  - [username.two] # Thursday
  - [username.one] # Friday
  - [] # Saturday
```
As the result of this configuration, `slack-duty-bot` will call `username.one` at Friday and full Slack group `some-group-name` at Saturday and Sunday

### Configuration priority

* Flags
* Environment variables
* Config file

## Deploy to Kubernetes

### Deploy with Helm

#### Configuration

The following table lists the configurable parameters of the slack-duty-bot chart and their default values.

| Parameter                         | Description                                | Default                                                   |
| --------------------------------- | ------------------------------------------ | --------------------------------------------------------- |
| `image.repository`                | SDB image registry                         | `iqoption/slack-duty-bot`                                 |
| `image.tag`                       | SDB Image tag                              | `{VERSION}`                                               |
| `image.pullPolicy`                | SDB image pull policy                      | `IfNotPresent`                                            |
| `configuration.slackToken`        | Slack token                                | `nil`                                                     |
| `configuration.keywords`          | Trigger words                              | array `duty`                                              |

#### Run deploy

```bash
helm upgrade --install slack-duty-bot-my-app-name .helm/slack-duty-bot/ --set configuration.slackToken=secret-token,configuration.keywords[0]="duty",configuration.keywords[1]="autobot",dutyUsers.monday=user.one,dutyUsers.tuesday=user.two,dutyUsers.wednesday=user.one
```

## Manual deploy to Kubernetes

### Create namespace

```bash
kubectl create namespace slack-duty-bot
```

### Create namespace quota

```yaml
#namespace-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: slack-duty-bot-quota
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 1Gi
    limits.cpu: "4"
    limits.memory: 2Gi
```

```bash
kubectl create -f namespace-quota.yaml --namespace=slack-duty-bot
```

### Create limit range

```yaml
#namespace-limit-range.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: slack-duty-bot-limit-range
spec:
limits:
  - default:
      cpu: "200m"
      memory: 128Mi
    defaultRequest:
      cpu: "100m"
      memory: 64Mi
    type: Container
```

```bash
kubectl create -f namespace-limit-range.yaml --namespace=slack-duty-bot
```

### Prepare your deployment file

```bash
(docker run \
    --rm \
    -it \
    -v $(pwd):/tmp \
    -e SDB_SLACK_TOKEN_BASE64=your-token-hash \
    -e SDB_NAME=your-deployment-name \
    -e SDB_TAG=1.1.0 \
    -e SDB_KEYWORD=your-keyword \
    -e SDB_SLACK_DEFAULT_USER=default-username \
    -e SDB_SLACK_GROUP_ID=group-id \
    -e SDB_SLACK_GROUP_NAME=group-name \
    supinf/envsubst /tmp/.kubernetes/deploy.yaml.tpl) > $(pwd)/.kubernetes/deploy.yaml
```

or use native `envsubst`

```bash
(SDB_SLACK_TOKEN_BASE64=your-token-hash \
    SDB_NAME=your-deployment-name \
    SDB_TAG=1.1.0 \
    SDB_KEYWORD=your-keyword \
    SDB_SLACK_DEFAULT_USER=default-username \
    SDB_SLACK_GROUP_ID=group-id \
    SDB_SLACK_GROUP_NAME=group-name \
    envsubst < $(pwd)/.kubernetes/deploy.yaml.tpl) $(pwd)/.kubernetes/deploy.yaml
```

After that you can change configuration with `kubect` or edit config map directly from Kubernetes dashboard

### Deploy

```bash
kubectl apply -f $(pwd)/.kubernetes/deploy.yaml --namespace slack-duty-bot
```

## Contributing

### Travis-CI and tests

To enable tests for your fork repository you **MUST**:

* Create your project in [TravisCI](http://travis-ci.com) for your fork repository
* Add environment variables to Travis-CI project:
  * `DOCKER_NAMESPACE`
  * `DOCKER_USER`
  * `DOCKER_PASSWORD`

Travis-CI will run test on every push for every ref and build docker image and push to [docker hub](http://hub.docker.io) **ONLY FOR TAGS**

## Changelog
[Changelog for project](CHANGELOG.md)

## Roadmap
[Roadmap for project](ROADMAP.md)

## Authors
* [Konstantin Perminov](https://github.com/SpiLLeR)
* [Ageev Pavel](https://github.com/insidieux)

## Contributors
* [Denis Arslanbekov](https://github.com/Arslanbekov)
