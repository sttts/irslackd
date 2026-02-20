# irslackd

[Slack ended IRC support][0] on May 15, 2018. So, we built our own Slack-IRC
gateway.

irslackd is actively developed and used daily on a 1000+ user Slack workspace.

[![Build Status](https://travis-ci.org/adsr/irslackd.svg?branch=master)](https://travis-ci.org/adsr/irslackd)

### Features

* TLS-encrypted IRCd
* Multiple Slack accounts/workspaces
* Channels, private channels, DMs, group DMs, threads
* Receive reactions, message edits, message deletes, attachments
* Proper en/decoding of @user, #channel, @team tags

### Setup

#### Using docker-compose

1. Clone irslackd and run docker-compose:
    ```
    $ git clone https://github.com/adsr/irslackd.git
    $ docker-compose up
    ```

    _Recommendation: Watch docker-compose build output for the generated certificate's fingerprint (used later for verification)._

1. Connect your IRC client to irslackd which listens on `127.0.0.1:6697`. See: [Configure your Slack account and IRC client](#configure-your-slack-account-and-irc-client)

#### Manual

1. [Install Node >=8.x][1] and npm. You can check your version of Node by
   running `node --version`.

2. Clone irslackd:
    ```
    $ git clone https://github.com/adsr/irslackd.git
    $ cd irslackd
    $ npm install    # Fetch dependencies into local `node_modules/` directory
    ```

3. Run `./bin/create_tls_key.sh` to create a TLS key and cert. This will put
   a private key and cert in `~/.irslackd`. Note the fingerprint.

4. Run irslackd:
    ```
    $ ./irslackd
    ```

    By default irslackd listens on `127.0.0.1:6697`. Set the command line
    options `-p <port>` and/or `-a <address>` to change the listen address.

### Configure your Slack account and IRC client

1. Follow the link below to obtain an irslackd token for your Slack workspace:

   [![Authorize irslackd](https://platform.slack-edge.com/img/add_to_slack.png)][2]
   
   Select the desired workspace in the dropdown in the upper right corner. Click
   'Authorize', and copy the access token. It will look something like this:

   `xoxp-012345678901-012345678901-012345678901-0123456789abcdef0123456789abcdef`

2. Connect to irslackd via your IRC client, e.g., WeeChat:
    ```
    /server add irslackd_workspace localhost/6697
    /set irc.server.irslackd_workspace.ssl on
    /set irc.server.irslackd_workspace.ssl_fingerprint fingerprint-from-step-3
    /set irc.server.irslackd_workspace.password access-token-from-step-5
    /connect irslackd_workspace
    ```
    Check the wiki for more [client configuration notes][5].

3. Repeat steps 1 and 2 for each Slack workspace you'd like to connect to.

4. Enjoy a fresh IRC gateway experience.

### OpenClaw integration (this fork)

This fork adds patches to use irslackd as an IRC backend for
[OpenClaw](https://openclaw.ai), an AI gateway. The changes enable
irslackd to work with browser-extracted Slack tokens (`xoxc-` + cookie)
without loading the full workspace user/channel lists on startup.

#### Connecting to Slack with a browser token

Instead of the OAuth app flow, you can extract tokens directly from a
logged-in Slack browser session:

1. Open Slack in your browser and press F12 to open DevTools.
2. Go to **Application → Storage → Cookies** and copy the value of the
   `d` cookie (starts with `xoxd-`). This is your session cookie.
3. In the browser **Console**, run:
   ```js
   JSON.parse(localStorage.localConfig_v2).teams[JSON.parse(localStorage.localConfig_v2).lastActiveTeamId].token
   ```
   Copy the result — it starts with `xoxc-`. This is your workspace token.
4. Combine them as the IRC password:
   ```
   xoxc-<token>|<d-cookie>
   ```

#### Running irslackd for OpenClaw (insecure/local mode)

```bash
./irslackd --insecure --port 6667 --channels ""
```

- `--insecure`: no TLS (safe when listening on localhost only)
- `--channels ""`: block all public Slack channels (only DMs and group
  DMs pass through); use `--channels "#general,#dev"` to allowlist
  specific public channels

#### Configuring OpenClaw

In `openclaw.json`, add an IRC channel pointing to irslackd:

```json
"irc": {
  "enabled": true,
  "host": "127.0.0.1",
  "port": 6667,
  "tls": false,
  "nick": "mybot",
  "password": "xoxc-TOKEN|COOKIE",
  "channels": ["#_openclaw"],
  "dmPolicy": "allowlist",
  "allowFrom": ["your-slack-nick"],
  "groupPolicy": "allowlist",
  "groupAllowFrom": ["your-slack-nick", "trusted-colleague"],
  "groups": {
    "#_openclaw": { "requireMention": false },
    "#mpdm-you--colleague--mybot26-1": { "requireMention": false },
    "*": { "requireMention": true }
  }
}
```

The `#_openclaw` channel is a virtual keepalive channel (no Slack
backend). OpenClaw must join it — this signals irslackd to start RTM.
Probe connections (OpenClaw status checks) never join `#_openclaw` and
therefore never trigger `rtm.connect`, avoiding Slack rate limits.

#### Access control

| Field | Description |
|-------|-------------|
| `dmPolicy: "allowlist"` | Only users in `allowFrom` can send direct messages. Use `"open"` + `allowFrom: ["*"]` to allow anyone (not recommended). |
| `allowFrom` | List of Slack nicks allowed to DM the bot. |
| `groupPolicy: "allowlist"` | Only group chats listed in `groups` (or matching `"*"`) are bridged. Blocks all other channels including public ones. |
| `groupAllowFrom` | List of Slack nicks allowed to send messages in any bridged group chat. |
| `groups` | Per-channel overrides. Use the exact IRC channel name (e.g. `#mpdm-alice--bob--mybot26-1`). `requireMention: false` means the bot responds to every message; `true` requires `@nick`. The `"*"` wildcard applies to all unlisted groups. |

**Safe defaults:** restrict DMs to yourself, list trusted users in `groupAllowFrom`, and require mention (`"*": { "requireMention": true }`) in group chats you don't fully control.

#### Patches in this fork

| Patch | Description |
|-------|-------------|
| Skip heavy init | `initUsers`/`initTeams`/`initChannels` skipped — too slow for large workspaces |
| Early welcome | Full `001`–`376` sequence sent immediately after `initialize()`, not waiting for RTM ready |
| Keep nick | Configured IRC nick is kept; Slack profile nick override disabled |
| `#_openclaw` keepalive | Virtual channel with no Slack backend; RTM starts on JOIN |
| Probe-safe presence | `setPresence('away')` only fires on disconnect if RTM was started |
| Self-mention translation | Own Slack user ID mapped to IRC nick so `<@USERID>` → `@nick` in messages |
| `--channels` allowlist | CLI flag to restrict which public Slack channels are bridged |

### Contribute

* Add more [client configuration notes][5].
* File bug reports and feature requests via [Github issues][3].
* Feel free to submit PRs. Make sure to include tests.

### Tests

* To run all tests: `npm test`
* To run a single test, e.g.: `npm test test_join`

### Related projects

* https://github.com/ltworf/localslackirc (another gateway, Python)
* https://github.com/insomniacslk/irc-slack (another gateway, Go)
* https://github.com/wee-slack/wee-slack (a terminal client, WeeChat-based)
* https://github.com/erroneousboat/slack-term (a terminal client, Go)
* https://github.com/42wim/matterircd (an IRCd for Mattermost and Slack)
* https://github.com/dylex/slack-libpurple (Slack plugin for libpurple)

### irslackd Slack workspace

* Feel free to join the [irslackd Slack workspace][4] for testing your
  irslackd setup.

[0]: https://my.slack.com/account/gateways
[1]: https://nodejs.org/
[2]: https://slack.com/oauth/authorize?client_id=2151705565.329118621748&scope=client
[3]: https://github.com/adsr/irslackd/issues
[4]: https://join.slack.com/t/irslackd/shared_invite/zt-5s6tvir7-Gp71YBznUVT5_z608xFQRg
[5]: https://github.com/adsr/irslackd/wiki/IRC-Client-Config
