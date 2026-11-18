# magpie-agentrouter

[English](README.md) · [中文](README.zh_CN.md)

A Magpie provider plugin for AgentRouter. It lets you use AgentRouter models in Magpie, read the account balance, and enable daily check-in.

## Features

- Sign in with an AgentRouter international-site or China-site account.
- View and use the AgentRouter models available to the account.
- Read the account balance.
- Enable daily check-in scheduled by Magpie.

## Install

Add this address on Magpie's plugin page:

```text
github:asterwyx/magpie-agentrouter
```

Or run this from a terminal that can reach Magpie:

```bash
magpie plugin add github:asterwyx/magpie-agentrouter
```

After installation, sign in on the AgentRouter provider page.

## Sign in

1. Choose **AgentRouter international site** or **China site**.
2. Fill in four separate fields:
   - AgentRouter account
   - Account password
   - API Key
   - System access token

The four credentials have different jobs and cannot replace one another:

- The account and password are used for sign-in and check-in.
- The API Key is used for the model list and model calls.
- The system access token is used for balance queries.

The site choice decides which AgentRouter site is used for models and sign-in; check-in also uses the selected site. Balance queries prefer the selected site and automatically try the other site when the selected one is temporarily unavailable.

You can also sign in from a terminal:

```bash
magpie plugin login agentrouter
```

OAuth, browser authorization, and Cookie sign-in are not supported.

## Usage

### Models

Choose an AgentRouter model in Magpie. Availability follows the account's actual model list; after a refresh in Magpie, use the updated list.

### Balance

View it on the provider page, or run:

```bash
magpie quota agentrouter
```

If the balance cannot be read, Magpie shows "remaining quota unknown". The plugin does not derive a balance from the password or API Key.

### Daily check-in

Enable AgentRouter daily check-in in Magpie. Magpie schedules it once a day; the plugin signs in with the account password and reads the site's check-in state. No separate local timer is needed.

## Optional setting

When AgentRouter rate-limits a request, the plugin reports a suggested wait time, 10 seconds by default. To change it, set `retryAfterSeconds` in Magpie's plugin options.

## FAQ

**No models are shown.**

Check that the API Key is correct. The model list follows what the account can actually use.

**The balance is unknown.**

The site endpoint may be temporarily unavailable, may return a verification page, or the system access token may need updating. Try again later; this does not necessarily mean the account is invalid.

**Sign-in fails.**

Check all four fields: account, password, API Key, and system access token. The account password and API Key are not interchangeable.

**Why does the provider appear as `agentrouter-plugin`?**

Magpie already has an older provider with the same ID. The plugin still works; remove or rename the old provider to restore the `agentrouter` prefix.

## Credential safety

The four credentials are stored on the server running Magpie, not only in the browser. Use them only on a trusted server and protect Magpie's data directory. Do not post credentials in chats, screenshots, issues, or Git repositories.

## Links

- [Contributing](CONTRIBUTING.md)
- [Changelog](CHANGELOG.md)
- [MIT License](LICENSE)
