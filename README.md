# Internet failover alerts for Home Assistant

[![Validate](https://github.com/SudoManaged/ha-internet-failover/actions/workflows/validate.yml/badge.svg)](https://github.com/SudoManaged/ha-internet-failover/actions/workflows/validate.yml)
![Home Assistant 2024.11+](https://img.shields.io/badge/Home%20Assistant-2024.11%2B-41BDF5)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

Get told when your home switches from its main internet connection to the backup (for example fiber to Starlink,
or cable to 4G/5G), and when it is back.

A free, open-source pair of Home Assistant blueprints by SudoManaged. It runs entirely on your own Home Assistant:
no add-ons, no custom integrations, no accounts, and no access to your router needed. It started as an alert we
use every day on our own Home Assistant, and was then generalized and tested for release.

## Contents

- [What is in it](#what-is-in-it)
- [Requirements and tested versions](#requirements-and-tested-versions)
- [How it works](#how-it-works)
- [Installation](#installation)
- [1. Get a public IP sensor](#1-get-a-public-ip-sensor)
- [2. Find your main connection's address](#2-find-your-main-connections-address)
- [3. Add the internet connection sensor](#3-add-the-internet-connection-sensor)
- [4. Add the alerts](#4-add-the-alerts)
- [5. Test it](#5-test-it)
- [What the alerts look like](#what-the-alerts-look-like)
- [Using the sensor in your own automations](#using-the-sensor-in-your-own-automations)
- [How fast is it?](#how-fast-is-it)
- [Troubleshooting](#troubleshooting)
- [Limitations](#limitations)
- [Updating and removing](#updating-and-removing)
- [Security and privacy](#security-and-privacy)
- [Reporting problems and contributing](#reporting-problems-and-contributing)
- [More free Home Assistant tools from SudoManaged](#more-free-home-assistant-tools-from-sudomanaged)
- [Changelog and license](#changelog-and-license)

## What is in it

| Blueprint | Type | What it does |
|---|---|---|
| [Internet connection (main or backup)](blueprints/template/sudomanaged/internet_connection.yaml) | template (a sensor) | A sensor whose state is the name of the connection in use, for example "Fiber" or "Starlink", worked out from your public IP address. It also keeps a "settled" connection, which follows only once a connection has been in use for a settle time. |
| [Internet failover alerts](blueprints/automation/sudomanaged/internet_failover_alerts.yaml) | automation | A notification when the settled connection switches to the backup, and when it is back on the main connection. |

The sensor is useful on its own too: on a dashboard, or to pause big uploads while you are on a metered backup
(see [Using the sensor in your own automations](#using-the-sensor-in-your-own-automations)).

## Requirements and tested versions

- **Home Assistant 2024.11 or newer.** Each blueprint declares this, and Home Assistant refuses to import the
  automation blueprint from the UI on an older release; a hand-copied blueprint file is not checked.
- **A main and a backup internet connection with automatic failover** in your router (dual-WAN routers, or a
  router with a cellular or satellite backup). The blueprints do not switch anything; they notice the switch.
- **The two connections must reach the internet from different public IP addresses.** They nearly always do,
  because they are different providers. See [Limitations](#limitations) for the exceptions.
- **A sensor with your public IP address.** Home Assistant's built-in DNS IP integration gives you one in a
  minute; see [1. Get a public IP sensor](#1-get-a-public-ip-sensor).
- The **logbook** integration (part of Home Assistant's default configuration).
- Both blueprints pass Home Assistant's configuration check on **2024.11.0**, **2026.3.3** and **2026.11.0**,
  and were run live on all three with a simulated public IP: an address already there when Home Assistant starts
  (picked up at start, no alert), a failed lookup in the middle of a switch (the sensor keeps the last connection),
  a switch to the backup (alert after the 1-minute settle time, and at once with a settle time of 0), garbage from
  the lookup (ignored), back on the main connection on an exact address (return alert; the bell cleared also with
  the return alert off), exact addresses versus longer ones and a YAML list of addresses, a 20-second flap (no
  alert), an unsteady return (main 30 s, backup 30 s, then main: one return alert), a renamed connection with a
  reload of template entities (no alert), a Home Assistant restart while on the backup (no repeated alert; the
  time on the backup still counted from the switch), a switch seen just before Home Assistant was stopped (alert
  at the first check after it started again), and a switch made while Home Assistant was down (alert once it had
  fully started, followed later by its "back on main" alert). The alerts were also tested on a real fiber and
  Starlink failover.
- Every change to this repository is checked automatically against Home Assistant 2024.11.0 and the current
  release (see [`.github/workflows/validate.yml`](.github/workflows/validate.yml)).

## How it works

When your router fails over, your home starts reaching the internet through the backup provider, and the rest of
the internet sees you at the backup provider's address. Home Assistant asks a public DNS service "what is my
address?" every couple of minutes (that is what the DNS IP integration does), so it sees the address change.

1. **The sensor** compares the address with your main connection's address (or the start of it). A match means
   the main connection; anything else means the backup. Its state is the name of the connection in use now. It
   reads the address whenever it changes, when Home Assistant starts, and every minute.
2. **During the switch itself** the lookup can fail for a moment. The sensor then keeps the last connection it
   knew, so a failed lookup is not mistaken for a switch.
3. **The settled connection** (an attribute of the sensor) follows the connection in use only once it has been in
   use for the settle time (1 minute by default; checked once a minute, so it follows 1 to 2 minutes after the
   switch is seen). A switch back and forth within that time, or a connection that flaps while it comes back,
   leaves it as it is.
4. **The alerts automation** notifies you whenever the settled connection changes between the main and the
   backup. So (with the return alert on) an "on the backup" alert is followed by one "back on main" alert, not
   several (see Limitations for the rare exceptions), and renaming a connection or restarting Home Assistant
   sends nothing. Because your home is already online again through the backup, the alert reaches your phone
   while the main connection is down.

```
  public IP sensor          connection now     settled (after 1 min)    alerts
  203.0.113.25  ───────▶   "Fiber"            "Fiber"
  (lookup fails) ──────▶   still "Fiber"      "Fiber"
  198.51.100.7  ───────▶   "Starlink"         "Starlink" after 1 min ─▶ "Internet on Starlink"
  203.0.113.25  ───────▶   "Fiber"            "Fiber" after 1 min    ─▶ "Internet back on Fiber"
```

## Installation

**The alerts blueprint** can be imported with this button (it opens your own Home Assistant through
my.home-assistant.io):

[![Import the Internet failover alerts blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FSudoManaged%2Fha-internet-failover%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fsudomanaged%2Finternet_failover_alerts.yaml)

**The sensor** is a *template* blueprint, which you use from `configuration.yaml`, so copy that file by hand. Or
copy both by hand, keeping the folder names:

| File in this repository | Copy to |
|---|---|
| `blueprints/template/sudomanaged/internet_connection.yaml` | `/config/blueprints/template/sudomanaged/internet_connection.yaml` |
| `blueprints/automation/sudomanaged/internet_failover_alerts.yaml` | `/config/blueprints/automation/sudomanaged/internet_failover_alerts.yaml` |

Then follow steps 1 to 5.

## 1. Get a public IP sensor

**With the DNS IP integration (recommended):** **Settings, Devices & services, Add integration, DNS IP**. Keep the
suggested hostname `myip.opendns.com` and submit. You get a sensor such as `sensor.myip` whose state is your public
IPv4 address, checked every 2 minutes. The lookup goes to OpenDNS's resolvers; nothing else is needed.

**Or in YAML, checked every minute** (if you prefer YAML or want a faster check), add a command line sensor to
`configuration.yaml` (under your existing `command_line:` list if you have one):

```yaml
command_line:
  - sensor:
      name: Public IP
      unique_id: public_ip_opendns
      command: "nslookup myip.opendns.com 208.67.222.222 2>/dev/null | awk '/^Name:/{n=1} n && /^Address/ {print $NF; exit}'"
      command_timeout: 15
      scan_interval: 60
```

That asks the same OpenDNS service with the `nslookup` tool that comes with the Home Assistant OS and container
images. After a restart you have `sensor.public_ip`.

Any other sensor whose state is your public IPv4 address works too.

## 2. Find your main connection's address

With the main connection working, open **Developer tools, States** and look at your public IP sensor: that is
your main connection's address, for example `203.0.113.25`.

- Always put it in quotes in YAML (`"203.0.113.25"`): unquoted, a start such as `81.10` would be read as the
  number 81.1.
- If your provider gives you a fixed address, use it as it is: `203.0.113.25` matches only 203.0.113.25.
- If it changes now and then, use the parts that stay the same, for example `203.0.113` (or `203.0.113.`), which
  matches every address from 203.0.113.0 to 203.0.113.255. Parts are always compared whole, so `203.0.11` does not
  match 203.0.113.25. Ask your provider, or note the address over a few days.
- Several addresses or starts, separated by commas, are allowed (for example a provider with two ranges).
- Ranges written as `203.0.113.0/24` are not understood (nothing would match, and every address would count as
  the backup); write the start, `203.0.113`, instead.

Everything that does not match counts as the backup, so you do not need the backup's addresses (satellite and
cellular providers change them often).

## 3. Add the internet connection sensor

Add this to `configuration.yaml` (if you already have a `template:` section, add the list item under it):

```yaml
template:
  - use_blueprint:
      path: sudomanaged/internet_connection.yaml
      input:
        ip_sensor: sensor.myip
        main_addresses: "203.0.113."
        main_name: Fiber
        backup_name: Starlink
        settle_minutes: 1
    name: Internet connection
    unique_id: internet_connection
```

| Setting | Default | Range | Explanation |
|---|---|---|---|
| `ip_sensor` | (required) | | The public IP sensor from step 1. |
| `main_addresses` | (required) | | Your main connection's address, or the start of it in whole parts; several separated by commas, or a YAML list (step 2). Quote it. |
| `main_name` | `Main` | | The name shown while on the main connection, for example `Fiber`. |
| `backup_name` | `Backup` | | The name shown while on the backup, for example `Starlink`. |
| `settle_minutes` | `1` | 0 to 30 | How long a connection must be in use before the settled connection (and so the alerts) follows it; checked once a minute, so it follows within a minute after that. A switch back within that time changes nothing. 0 follows at once. |

Keep the `unique_id`: it lets you manage the sensor in the UI (name, icon, area). Check the configuration
(**Developer tools, YAML, Check configuration**) and restart Home Assistant. You now have
`sensor.internet_connection`. Right after Home Assistant starts it shows your main connection's name (or
**unknown** until the public IP sensor has its first address; it looks again every minute).

Its attributes:

| Attribute | Meaning |
|---|---|
| `on_backup` | `true` while on the backup connection now. |
| `public_ip` | The last public address seen. |
| `since` | When the connection now last switched between the main and the backup (or was first seen). |
| `settled` | The settled connection's name. |
| `settled_backup` | `true` while the settled connection is the backup. |
| `settled_since` | When the connection that is now settled started (the switch time, not the end of the settle time). |
| `startup_wait` | `0` normally; `2` from a Home Assistant shutdown or start until it has fully started (automations are listening by then). The settled connection changes only at `0`. Each once-a-minute check also counts it down, so it never stays above `0`. |

## 4. Add the alerts

From the UI: **Settings, Automations & scenes, Create automation, Create new automation from a blueprint**,
choose **Internet failover alerts**. Make one automation per place an alert should go (for example one for your
phone and one for email).

How long to wait before alerting is the sensor's settle time (step 3).

| Setting | Default | Range | Explanation |
|---|---|---|---|
| Internet connection sensor | (required) | | The sensor from step 3. |
| Also alert when the main connection is back | on | | A second alert when the main connection is back, with how long you were on the backup. |
| Notify action | `notify.notify` | | Where alerts go. For one phone use `notify.mobile_app_<your_phone>`; with the Home Assistant app installed, `notify.notify` sends to every phone that has the app. Notify groups defined in YAML work too. Notify *entities* (`notify.send_message`, including notify groups made in the UI) are not supported. |
| Home Assistant app options | on | | One tag, so "back on main" replaces "on backup" on your phone, and high priority with `ttl: 0`, so Android delivers it at once (iOS ignores these two). Turn it off if your notify action is not the Home Assistant app. |
| Also show it inside Home Assistant | on | | A Home Assistant notification (the bell) while you are on the backup, dismissed when the main connection is back (also when "Also alert when the main connection is back" is off). All automations for the same sensor share one bell notification, so it is enough to leave this on in one of them. |

## 5. Test it

Unplug your main connection at the router (or switch it off), so the router fails over. Within about 2 minutes
plus the settle time (with the DNS IP integration; about 1 minute plus the settle time with the YAML sensor) you
get "Internet on Starlink". Plug it back in, and after the same time you get "Internet back on Fiber".

Watching it happen: **Developer tools, States**, `sensor.internet_connection`, and the **Logbook**, where every
alert is written under "Internet failover".

## What the alerts look like

With the names `Fiber` and `Starlink`:

- **"Internet on Starlink"**: "Fiber is down: your home switched to Starlink at 2:14 PM (public IP 198.51.100.7)."
- **"Internet back on Fiber"**: "Fiber has been back since 2:51 PM (public IP 203.0.113.25), after 37 min on
  Starlink."

The time shown is when the switch happened (not when the settle time ended), in Home Assistant's time zone and
12-hour format. The time spent on the backup is rounded to the nearest minute and shown as "less than a minute",
"N min" or "N h M min"; it counts from the switch, also across Home Assistant restarts.

## Using the sensor in your own automations

Pause a big cloud backup while on a metered backup connection (it follows the settled connection, so a short
blip does not pause anything):

```yaml
triggers:
  - trigger: template
    value_template: "{{ is_state_attr('sensor.internet_connection', 'settled_backup', true) }}"
    id: backup
  - trigger: template
    value_template: "{{ is_state_attr('sensor.internet_connection', 'settled_backup', false) }}"
    id: main
actions:
  - if:
      - condition: trigger
        id: backup
    then:
      - action: switch.turn_off
        target:
          entity_id: switch.nas_cloud_backup
    else:
      - action: switch.turn_on
        target:
          entity_id: switch.nas_cloud_backup
```

Each trigger only fires when its condition becomes true, so a reload of template entities (when the attributes
are briefly missing) at most repeats the action that already applies.

Show it on a dashboard with a tile card:

```yaml
type: tile
entity: sensor.internet_connection
name: Internet
```

## How fast is it?

- The DNS IP integration checks every 2 minutes, the YAML sensor in step 1 every minute.
- The settled connection then follows after the settle time, at the next once-a-minute check: 1 to 2 minutes
  with the default of 1 minute.
- So with the defaults you hear 1 to 4 minutes (DNS IP) or 1 to 3 minutes (YAML sensor) after the switch. If a
  lookup fails during the switch, add a check or two: the DNS IP integration keeps showing the
  old address through two failed lookups. A switch back and forth within that time may not be seen at all, which
  is usually what you want.
- After a Home Assistant restart, the settled connection waits until Home Assistant has fully started
  (automations are listening by then). A switch made while Home Assistant was down or restarting is announced
  at the first once-a-minute check after that, once it has lasted the settle time.

## Troubleshooting

- **The sensor stays unknown.** The public IP sensor has no address yet. Open it in **Developer tools, States**:
  if it is unknown, unavailable or empty, the lookup does not work from your Home Assistant (a firewall blocking
  DNS to OpenDNS, for example); try the other method in step 1.
- **It says backup while the main connection is working.** Your main address changed and no longer matches. Look
  at the `public_ip` attribute and widen `main_addresses` (use the start of the address, step 2).
- **No alert on a switch.** Check that the sensor changed (its history) and that its `settled` attribute followed
  (the new connection must last longer than the settle time). Then open the automation's trace.
- **An alert, but no phone notification.** Open the automation's trace: it shows the exact error, for example a
  notify action that does not exist.
- **Both alerts pile up on the phone.** Turn on "Home Assistant app options", and make sure the notify action
  reaches the Home Assistant app.

## Limitations

- **It tells you about a switch, not about an outage.** If both connections are down, the public address cannot
  be looked up, so the sensor keeps showing the last connection and no alert is sent (it could not reach your
  phone anyway).
- **The two connections must leave from different public addresses.** If the main connection and the backup come
  from the same provider and share addresses, or if Home Assistant reaches the internet through a VPN, the public
  address does not tell them apart.
- **IPv4 only.** The sensor compares IPv4 addresses (what the DNS IP integration and the YAML sensor report).
- It notices the switch, not the cause: an alert means your home is on the backup, for whatever reason.
- On Home Assistant 2026.8 and later, after a power cut or crash (not a normal restart or shutdown), a switch
  seen just before it (or any switch, with a settle time of 0) can settle in the first half-minute of the start,
  before automations listen, and is then not announced; the sensor itself is right. A switch made while Home
  Assistant was down is not affected with a settle time of 1 minute or more.
- The alerts automation must be on to see a change: one made while it is off or being reloaded is not
  announced. And if the sensor is first set up while you are on the backup, it starts there without an alert,
  so the first alert you get is "back on main".
- Home Assistant clears its own notifications (the bell) when it restarts, so the "on the backup" notification
  disappears at a restart even if you are still on the backup; the phone notification stays.
- The sensor's settings live in `configuration.yaml`; Home Assistant has no UI for template blueprints.

## Updating and removing

- **Updating:** replace the blueprint files with the new versions and restart Home Assistant (for the alerts
  blueprint, **Re-import blueprint** in the blueprint's menu also works).
- **Removing:** delete the alerts automations, delete the `template:` entry that uses the sensor blueprint, and
  restart. Because the sensor had a `unique_id`, its entity stays listed under **Settings, Devices & services,
  Entities** as unavailable; delete it there. Then delete the blueprint files (and the public IP sensor if you no
  longer need it).

## Security and privacy

- The blueprints open nothing to the network and use no webhooks.
- Your public address is looked up with OpenDNS (by the DNS IP integration or the YAML sensor), which sees that
  your home asked; nothing else leaves your Home Assistant except the notifications you configure, which include
  your public address.
- To report a security problem privately, use **Report a vulnerability** on this repository's
  [Security tab](../../security). Please do not open a public issue for security problems. See
  [SECURITY.md](SECURITY.md).

## Reporting problems and contributing

- **Something does not work:** open an [issue](../../issues) with your Home Assistant version, your kind of main
  and backup connection, what you expected and what happened. The sensor's history and an automation trace help a
  lot; remove your public addresses from them first.
- **Improvements:** pull requests are welcome. Keep the blueprints self-contained (no custom integrations) and
  documented in this README.

## More free Home Assistant tools from SudoManaged

- [Webhook alerts](https://github.com/SudoManaged/ha-webhook-alerts)
- [Bluetooth self-heal](https://github.com/SudoManaged/ha-bluetooth-self-heal)
- [Yealink desk phone call log](https://github.com/SudoManaged/ha-yealink-call-log)
- [Dehumidifier kit](https://github.com/SudoManaged/ha-dehumidifier-kit)
- [Daily schedule from several calendars](https://github.com/SudoManaged/ha-daily-schedule)

## Changelog and license

See [CHANGELOG.md](CHANGELOG.md). Released under the [MIT License](LICENSE). Copyright (c) 2026 SudoManaged.
