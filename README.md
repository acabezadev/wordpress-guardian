# wordpress-guardian

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

Small file-integrity monitor for WordPress installs. It watches the web root with inotify and sends a Telegram message when a file is created, modified, deleted, moved or has its attributes changed. Single C++17 binary, libcurl is the only dependency.

## What it does

- Watches one or more directories recursively (every subdirectory present at startup gets its own inotify watch).
- Reacts to `IN_MODIFY`, `IN_CREATE`, `IN_DELETE`, `IN_ATTRIB`, `IN_MOVED_FROM` and `IN_MOVED_TO`.
- Ignores dotfiles, names containing `~`, WordPress' `temp-write-test` probe and Wordfence's `wflogs` directory.
- Queues events and sends them as one Telegram message every 5 minutes, so an attacker dropping fifty files produces one alert, not fifty.

## Why

WordPress sites on a shared VPS get hit through outdated plugins, and the first visible symptom is usually a new or modified PHP file. Hash-based scanners run on a schedule and are heavy; this is the opposite: a daemon that costs nothing while idle and tells you within minutes that something touched the tree. It is a tripwire, not a replacement for backups, updates or a WAF.

## Install

Requires Linux (inotify), a C++17 compiler, CMake 3.26+ and the libcurl development package.

```bash
# Debian / Ubuntu
sudo apt install build-essential cmake libcurl4-openssl-dev

git clone https://github.com/acabezadev/wordpress-guardian.git
cd wordpress-guardian
cmake -B build
cmake --build build
# binary: build/wordpress_guardian
```

## Usage

Edit the configuration (see below), build, then run the binary as a user that can read the watched directories. It runs in the foreground and logs to stdout, so the practical way to keep it alive is a systemd unit:

```ini
# /etc/systemd/system/wordpress-guardian.service
[Unit]
Description=wordpress-guardian file-integrity monitor
After=network-online.target

[Service]
ExecStart=/usr/local/bin/wordpress_guardian
Restart=on-failure
User=www-data

[Install]
WantedBy=multi-user.target
```

```bash
sudo install -m 0755 build/wordpress_guardian /usr/local/bin/
sudo systemctl enable --now wordpress-guardian
journalctl -u wordpress-guardian -f
```

If the watched tree is large, raise the inotify limit: `sudo sysctl fs.inotify.max_user_watches=524288`.

## Configuration

Configuration is compile-time: three constants at the top of `main.cpp`.

| Constant | What it is |
| --- | --- |
| `FOLDERS_TO_WATCH` | Array of absolute paths to watch recursively, e.g. `/var/www/example.com`. |
| `TELEGRAM_BOT_TOKEN` | Token of the bot that sends the alerts (create one with [@BotFather](https://t.me/BotFather)). |
| `TELEGRAM_CHAT_ID` | Chat or group ID the alerts go to. |

Keep the token out of version control: build from a local copy of `main.cpp` or inject it at build time. Never commit a binary that embeds a real token.

## How it works

One `inotify` instance, one watch descriptor per directory, a map from descriptor to path to rebuild the full path of each event. Events go into a mutex-protected queue; a second thread wakes up every 5 minutes, concatenates whatever is queued and POSTs it to `https://api.telegram.org/bot<token>/sendMessage` with libcurl. About 150 lines, no framework.

## Known limitations

- Directories created after startup are not added to the watch set (restart the service after deploying new plugins or themes).
- The Telegram request is a plain POST and the message text is not URL-encoded, so paths containing `&`, `#` or `+` may arrive truncated.
- No retry or error handling on network or Telegram API failures: a failed batch is lost.
- Linux only.

## Security

This tool is a detection aid. It does not block anything, does not verify file contents and can be silenced by anyone who can kill the process or modify the binary. Run it as a dedicated user, keep the binary outside the web root and treat the Telegram token as a secret.

To report a vulnerability in this project, see [SECURITY.md](SECURITY.md).

## Status

Stable. Built in 2023 for a handful of WordPress sites on a VPS. Maintained on a best-effort basis; issues and small pull requests are welcome.

## License

MIT. See [LICENSE](LICENSE).
