# Akamai CLI: Enterprise Application Access<!-- omit in toc -->

## Table of contents<!-- omit in toc -->

- [Screenshot](#screenshot)
- [Introduction](#introduction)
- [Key features](#key-features)
- [Installation](#installation)
  - [Configuration file](#configuration-file)
  - [Upgrade cli-eaa](#upgrade-cli-eaa)
- [Examples](#examples)
  - [EAA event logs](#eaa-event-logs)
  - [Applications](#applications)
  - [Directory operations](#directory-operations)
  - [Connectors](#connectors)
- [Known Limitations](#known-limitations)
- [Troubleshooting and Support](#troubleshooting-and-support)
  - [Self-troubleshooting](#self-troubleshooting)
  - [Support](#support)

## Screenshot

<img src="docs/cli-eaa-terminal@2x.png" width="40%" />

## Introduction

[Enterprise Application Access (EAA)](https://www.akamai.com/us/en/products/security/enterprise-application-access.jsp) comes with a full suite of APIs. 
Yet you need to write scripts or use [Postman](https://developer.akamai.com/authenticate-with-postman) to be able to interact with the service.

With [Akamai CLI](https://developer.akamai.com/cli) you can run very common operations directly from the command line, no coding required. 

This can be helpful if you plan to consume EAA logs into your favorite SIEM, or automate some operation in your workflow with Bash, Powershell or solution like Ansible.

## Key features

- Event logs
  - View access logs (identification, application activity)
  - View admin logs (admin portal access, config change, deployment, deletion)
  - Send the logs to a file
  - Blocking mode (similar to `tail -f`)
  - Alternatively, you can specify a date range with `--start` and `--end`
- Application
  - Save, restore/update, deploy
  - Batch operation
- Directory
  - Create group and group overlay
  - Synchronize with your LDAP or Active Directory
- Connectors
  - List all connectors including the reachability status

## Installation

Make sure your first have Akamai CLI installed on your machine.

We support a wide variety of platform: Windows, Mac, Linux, container...

For more information, please visit the [Getting Started](https://developer.akamai.com/cli/docs/getting-started) guide on developer.akamai.com.

Once the Akamai CLI is installed, the `cli-eaa` module installation is done via `akamai install eaa` command:

```
$ akamai install eaa
```

And voilà!

The command takes care of all the dependencies. 

To check your cli-eaa version with the `version` command

```
$ akamai eaa version
0.2.2
```

### Configuration file

In order to work, the CLI module will look for an `.edgerc` configuration file stored 
in your home directory or your prefered location. \
For the latter make sure to use the `--edgerc` parameter in the command line.\

To create a {OPEN} API user, follow [these instructions](https://developer.akamai.com/legacy/introduction/Prov_Creds.html).
Make sure the API user has READ-WRITE permission to *Enterprise Application Access*.

To create a legacy API key and secret from, connect to Akamai Control Center. 
- use Enterprise Application Access in the left menu
- go to **System** > **Settings** and 
- then click **Generate new API Key** in the **API** section of the page

The `.edgerc` file should look like:

```INI
[default]

; EAA Legacy API used by the 'akamai eaa log' command
eaa_api_host = manage.akamai-access.com
eaa_api_key = XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXXX
eaa_api_secret = XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXXX

; {OPEN} API for everything else
host = akaa-xxxxxxxxxxxxxxxx-xxxxxxxxxxxxxxxx.luna.akamaiapis.net
client_token = akab-xxxxxxxxxxxxxxxx-xxxxxxxxxxxxxxxx
client_secret = xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
access_token = akab-xxxxxxxxxxxxxxxx-xxxxxxxxxxxxxxxx
; If your organization have multiple contracts with EAA service
; please add it below. Contact your Akamai representative to obtain it
contractid = A-B-1CD2E34
```

### Upgrade cli-eaa

To upgrade your CLI EAA module to the latest version, use:

```bash
$ akamai update eaa
```

## Examples

### EAA event logs

EAA has two types of logs, the user access logs and the administrators audit logs.
For detailed description about each field, please refer to the product documentation on [https://learn.akamai.com](https://learn.akamai.com/en-us/webhelp/enterprise-application-access/eaa-logs-from-eaa-api-and-splunk/GUID-07D6B02C-1EDE-4D16-A19D-687449B4A748.html).

You can pull log either in near realtime, using `-f`, or retrieve a period of time passing EPOCH timestamp in `--start` and `--end`
You cannot combine `-f` and explicit date range.

Pull user access logs, block till new logs are received.
You can stop by pressing Control+C (Control+Break) or sending a signal SIG_INT or SIG_TERM to the process

```bash
$ akamai eaa log access --tail
```

You may want a one time chunk of log for a period of time, let's say the last 6 hours:

```bash
$ START=$(bc <<< "$(date +%s) - 6 * 60 * 60")
$ akamai eaa log access -s $START
```

Send the **user access events** to a file (utf-8 encoding is being used):
```bash
$ akamai eaa log access --tail -o /tmp/eaa_access.log
```

Pull **admin audit events**, block till new logs are received
```bash
$ akamai eaa log admin --tail
```

### Applications

```
$ akamai eaa search datascience
app://mD_Pw1XASpyVJc2JwgICTg,Data Science,akdemo-datascience,akdemo-datascience.go.akamai-access.com,4
Found 1 app(s), total 124 app(s)
```

You can save locally the application
```
$ akamai eaa app app://mD_Pw1XASpyVJc2JwgICTg > ~/eaa_app_datascience_v3.json
```

And restore
```
$ cat ~/eaa_app_datascience_v3.json | akamai eaa app app://mD_Pw1XASpyVJc2JwgICTg update
```

Or quickly walk through the JSON tree with `jq`.
```
$ akamai eaa -b app app://mD_Pw1XASpyVJc2JwgICTg | jq .advanced_settings.websocket_enabled
"true"
```

Delete an application
```
akamai eaa app app://mD_Pw1XASpyVJc2JwgICTg delete
```

One question we often get: *What are the applications using connector `xyz`?*\
Buckle up, we use `jq` and `grep`.\
Note: we use `-b` to avoid the extra info the CLI spills out, like the footer.

```
$ akamai eaa -b search | akamai eaa app - | jq -j '.name, ": ", (.agents[]|.name, " "), "\n"'|grep xyz
```

View groups associated with a particular application
```
$ akamai eaa app app://FWbUCfpvRKaSOX1rl0u55Q viewgroups
```

You can pipe command as well, example to deploy all the application matching "tunnel"

```
$ akamai eaa -b search bastion | akamai eaa app - deploy
```

### Directory operations

List the configured directories

```
$ akamai eaa dir
dir://FuiibQiDQzmC34oBx7INfQ,Cloud Directory,7
dir://2Kz2YqmgSpqT_IJq9BLkWg,ad.akamaidemo.net,108
dir://EX5-YjMyTrKgeWKHrqhUEA,Okta LDAP,10
dir://Ygl1BpAFREiHrA8HR7dFhA,Azure AD,1
```

Trigger directory synchronization

```
$ akamai eaa dir dir://2Kz2YqmgSpqT_IJq9BLkWg sync
Synchronize directory 2Kz2YqmgSpqT_IJq9BLkWg
Directory 2Kz2YqmgSpqT_IJq9BLkWg synchronization requested.
```

### Connectors

Here with the shortcut `c` and the `column` command available in most POSIX environment.
When piping, the extra information written on *stderr* so they appear seperately.

```
$ akamai eaa c | column -t -s,
Total 9 connector(s)
#Connector-id                 name                reachable  status  version     privateip      publicip        debug
con://cht3_GEjQWyMW9LEk7KQfg  demo-v2-con-1-amer  1          1       4.4.0-2765  10.1.4.206     12.123.123.123  Y
con://Wy0Y6FrwQ66yQzLBAInC4w  demo-v2-con-2-amer  1          1       4.4.0-2765  10.1.4.172     12.123.123.123  Y
con://dK0f1UvhR7i8-RByABDXaQ  demo-v2-con-4-emea  1          1       4.4.0-2765  192.168.1.90   12.123.12.12    N
con://Ihmf51dASo-R1P37hzaP3Q  demo-v2-con-3-emea  1          1       4.4.0-2765  192.168.1.235  12.123.12.12    N
con://XiCmu80xQcSWnaeQcvH8Vg  demo-v2-con-5-apj   1          1       4.4.0-2765  192.168.1.228  12.123.123.12   Y
con://pkGjL5OgSjyHoymMguvp9Q  demo-v2-con-6-apj   1          1       4.4.0-2765  192.168.1.144  12.123.123.12   Y
con://NAWSlptPSXOjq-bk2-EQPw  demo-v2-con-10-rus  1          1       4.4.0-2765  10.3.0.101     12.123.123.12   Y
con://e_0nShZBQ7esNAC3ZEkhSQ  demo-v2-con-3-amer  1          1       4.4.0-2765  10.1.4.83      12.123.123.123  Y
con://OEe9o-n2S_aMeZpLxgwG0A  tmelab-sfo          1          1       4.4.0-2765  192.168.2.101  12.123.123.12   Y
```

## Known Limitations

- Akamai CLI `--proxy` option is not supported
- While updating an application from a JSON, only a subset of the data will be updated in the back-end, not the entire application configuration

## Troubleshooting and Support

### Self-troubleshooting

If the command is not working properly, you can increase the level of verbosity using:

- `-v` or `--verbose` to trace the main steps
- `-d` or `--debug` to get full visibility, include API HTTP headers

The messages are printed on _stderr_ so you can safely redirect stdout to a file or use the `--output` option.

### Support

`cli-eaa` is provided as-is and it is not supported by Akamai Support.
To report any issue, feature request or bug, please open a new issue into the [GitHub Issues page](https://github.com/akamai/cli-eaa/issues)

We are strongly encouraging developer to create a pull request.
