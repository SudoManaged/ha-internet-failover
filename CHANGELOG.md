# Changelog

All notable changes. Versions follow [Semantic Versioning](https://semver.org/).

## 1.0.0 (2026-11-05)

First public release.

- **Internet connection (main or backup)** (template blueprint): a sensor whose state is the name of the
  connection in use, from your public IP address compared with your main connection's address or its start (in
  whole parts; a comma-separated text or a YAML list); keeps the last connection while the address cannot be
  looked up; reads the address when it changes, when Home Assistant starts and every minute. A "settled"
  connection follows only once a connection has been in use for the settle time, so flaps change nothing; after
  a start it waits until Home Assistant has fully started, so a switch during a restart is still announced.
- **Internet failover alerts** (automation blueprint): a notification when the settled connection switches to
  the backup and when the main connection is back (with the time spent on the backup, counted across restarts);
  with the return alert on, an "on the backup" alert is followed by one "back on main" alert; renaming a
  connection, a reload or a restart sends nothing; Home Assistant app options and a Home Assistant notification
  while on the backup, always cleared on the way back.
- Tested live on Home Assistant 2024.11.0, 2026.3.3 and 2026.11.0.
