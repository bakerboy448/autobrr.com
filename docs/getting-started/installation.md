---
sidebar_position: 1
---

# Installation

:::tip

The easiest way to install it is via [swizzin.ltd/applications/autobrr](https://swizzin.ltd/applications/autobrr).

:::

But if you don't run Swizzin you can of course still use it, with some setup.

Follow instructions below for recommended setup on a regular linux server. Separate Docker instructions are coming.

## Download package

Download the latest release, or download source and build yourself.

Check [latest releases](https://github.com/autobrr/autobrr/releases/latest) and download the one for your system.

**Check for latest version!**

```bash
wget https://github.com/autobrr/autobrr/releases/download/v0.9.0/autobrr_0.9.0_linux_x86_64.tar.gz
```

Unpack. Run with `root` or `sudo`. If you don't have root or are on a shared system, place the binaries somewhere in your home dir like `~/.bin`

```bash
tar -C /usr/local/bin -xzf autobrr_0.9.0_linux_x86_64.tar.gz
```

This will extract both `autobrr` and `autobrrctl` to `/usr/local/bin`.


## Create config

Create a config dir in your users home dir like `~/.config/autobrr`

```bash
mkdir -p ~/.config/autobrr && touch ~/.config/autobrr/config.toml
```

Add config

```toml
# config.toml

# Hostname / IP
#
# Default: "localhost"
#
host = "127.0.0.1"

# Port
#
# Default: 8989
#
port = 8989

# Base url
# Set custom baseUrl eg /autobrr/ to serve in subdirectory.
# Not needed for subdomain, or by accessing with the :port directly.
#
# Optional
#
#baseUrl = "/autobrr/"

# autobrr logs file
# If not defined, logs to stdout
#
# Optional
#
#logPath = "log/autobrr.log"

# Log level
#
# Default: "DEBUG"
#
# Options: "ERROR", "DEBUG", "INFO", "WARN"
#
logLevel = "TRACE"

# Session secret
#
sessionSecret = "secret-session-key"
```

### Config options

* `host`: If not using a reverse proxy, change to `0.0.0.0`.
* `port`: If port already in use then change to a free one.
* `baseUrl`: **`OPTIONAL`** It supports running on both the root url and in a subpath, as well as subdomain. Uncomment if needed.
* `logPath`: **`OPTIONAL`** It can be useful to log to file. If running with systemd you can use `journalctl` to check logs. It has built in logrotation to not fill up disk.
* `logLevel`: Choose how much log output you want to see. Needs a restart to take effect.
* `sessionSecret`: Used for session cookies. Change to something more random like a `UUID`.

## Create user

To create the initial database and a user, you need to use `autobrrctl`. Point `--config` to your config directory, and set username to your desired one. It will ask for password.

```bash
autobrrctl --config ~/.config/autobrr create-user USERNAME
```

## Systemd

This is the recommended way to run autobrr on linux based systems.

Create a service file in `/etc/systemd/system/` called `autobrr.service`.

```bash
touch /etc/systemd/system/autobrr.service
```

Then fill it with the following service definition

```systemd title="/etc/systemd/system/autobrr.service"
[Unit]
Description=autobrr service for %i
After=syslog.target network.target
[Service]
Type=simple
User=%i
Group=%i
ExecStart=/usr/bin/autobrr --config=/home/%i/.config/autobrr/
[Install]
WantedBy=multi-user.target
```

Start the service. Enable will make it startup on reboot.

```bash
systemctl enable -q --now autobrr
```

## Done

Now it's up and running and you should be able to visit it at your `domain.ltd:8989` and login. Check next pages for further setup.

## Reverse proxy

It's recommended to run it behind a reverse proxy like nginx to get TLS and all that good stuff.