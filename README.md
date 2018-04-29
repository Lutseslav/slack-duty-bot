# Slack duty bot
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Go Report Card](https://goreportcard.com/badge/github.com/iqoption/slack-duty-bot)](https://goreportcard.com/report/github.com/iqoption/slack-duty-bot)

### How usage
1. Create new custom integration `Bots` (e.g https://{team}.slack.com/apps/manage/custom-integrations)
2. Build for your environment
3. Prepare config.yaml with duties list
3. Run with the required parameters

```bash
SDB_SLACK_TOKEN=your-token-here ./slack-duty-bot \
    --slack.keywords keyword-1 \
    --slack.keywords keyword-2 \
    --slack.group.id your-group-id \
    --slack.group.name your-group-name
```

### Build package
Build
```bash
go get -u github.com/golang/dep/cmd/dep
dep ensure
env GOOS=linux GOARCH=amd64 go build -v
```
Build via makefile
```bash
make BUILD_OS=linux BUILD_ARCH=amd64
```
Build in docker
```bash
docker run --rm -v $(pwd):/go/src/slack-duty-bot -w /go/src/slack-duty-bot golang:1.10 make BUILD_OS=linux BUILD_ARCH=amd64
```

### Configuration flags, environment variables
Environment variables are prefixed with `SDB_` and *MUST* be uppercase with `_` delimiter
Available variables:
* `SDB_CONFIG_PATH`
* `SDB_SLACK_TOKEN`
* `SDB_SLACK_GROUP_ID`
* `SDB_SLACK_GROUP_NAME`
* `SDB_SLACK_THREADS`

Every environment variable can be overwritten by startup flags
Available flags:
* `--config.path` - path to yml config (default: . and $HOME/.slack-duty-bot)
* `--slack.token` - Slack API client token
* `--slack.keywords` - Case insensitive keywords to search in message text, can be set multiple times (default: [])
* `--slack.group.name` - Slack user group name, to mention in channel if duty list is empty
* `--slack.group.id` - Slack user group ID, to mention in channel if duty list is empty
* `--slack.threads` - Case insensitive keywords to search in message text, can be set multiple times (default: true) 

You can get IDS from api or just use [testing page](https://api.slack.com/methods/usergroups.list/test)

### Configuration file
Configuration file *MUST* contain `duties` key with *7* slices of Slack username
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
