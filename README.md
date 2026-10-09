# integrity-watch

A simple intrusion watch for Debian and Ubuntu servers, in a single bash script.

It takes a snapshot of the things attackers usually change and compares it with a baseline you
took while the server was known to be clean. If anything deviates, you find out, ideally the same
day rather than months later.

The script was written after an intrusion on temperatur.nu in the autumn of 2026. The attacker
stayed for almost three months, with a backdoor disguised as a system service, a scheduled job
under the web server's account, a replaced system file and a crypto miner in a temp directory.
This check would have flagged every one of those within an hour.

## What it watches

| Area | What is checked | Alert |
|---|---|---|
| `dpkg` | Files that belong to a package but no longer match the package's checksum | Always CRIT |
| `tmpexec` | Executable files in `/tmp`, `/var/tmp`, `/dev/shm` | Always CRIT |
| `procs` | Processes whose binary has been deleted or lives in a temp directory | Always CRIT |
| `units` | Enabled systemd units, and unit files not owned by any package | New = CRIT, gone = WARN |
| `cron` | Every cron source including each user's crontab, plus systemd timers | New/changed = CRIT (package-owned file = WARN) |
| `shells` | Accounts that can log in | New = CRIT |
| `suid` | setuid/setgid files | New = CRIT |
| `listen` | Listening ports and the program behind each | New = CRIT |
| `sshkeys` | The contents of every `authorized_keys` | Changed = CRIT |

Regular security updates do not cause false alarms: `dpkg -V` compares against the package that
is installed right now, and changed cron files owned by a package only raise a WARN.

## What it does not do

It is not a complete intrusion detection system. A skilled attacker with root can hide from
anything running on the same machine, including this script. It catches the common case:
automated intrusions that drop backdoors, miners and scheduled jobs. For thorough file integrity
monitoring, look at [AIDE](https://aide.github.io/), and consider combining it with `debsums`,
`rkhunter` and shipping logs to another machine.

## Installation

```sh
sudo install -m 755 integrity-watch /usr/local/sbin/integrity-watch
sudo integrity-watch baseline        # while the server is known to be clean
sudo integrity-watch check           # compare now
```

Add a check to root's crontab (`sudo crontab -e`), for example hourly with an email on
deviations:

```
17 * * * * nice -n19 ionice -c3 /usr/local/sbin/integrity-watch check --mail you@example.com >/dev/null
```

The email requires a working `mail` command on the server (for example `bsd-mailx` with Postfix).
The exit code follows the Nagios convention (0 OK, 1 WARN, 2 CRIT, 3 UNKNOWN), so the script can
also be used directly as a Nagios or Icinga check.

## After deliberate changes

When you have added a service, a cron job or an ssh key, or opened a port, take a new baseline:

```sh
sudo integrity-watch baseline
```

Run `check` first and read through what changed, so you don't approve something you didn't do
yourself.

## Settings

| Environment variable | Default | Meaning |
|---|---|---|
| `INTEGRITY_WATCH_DIR` | `/var/lib/integrity-watch` | Where the current snapshot and baseline are stored |
| `INTEGRITY_WATCH_DPKG_IGNORE` | doc/man/locale + PEAR registry | Regex for package files allowed to differ |

## Example output

A run that caught a planted executable and a new crontab for `www-data`:

```
integrity-watch CRIT on www 2026-10-08 22:03:11
baseline: 2026-10-08T20:41:07+02:00 host=www
OK    dpkg
CRIT  tmpexec: 1 finding(s)
        /var/tmp/.x www-data 2803712
OK    procs
OK    units
CRIT  cron: +1 / -0 vs baseline
      + user:www-data 4f1c0e2a9b3d
OK    shells
OK    suid
OK    listen
OK    sshkeys
```

## License

MIT. Use, modify and share freely.
