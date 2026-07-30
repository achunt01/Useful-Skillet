# PAN-OS Upgrades via CLI

CLI procedure for upgrading PAN-OS on standalone firewalls and active/passive HA pairs. Examples target the **11.1** line. The CLI is worth preferring over the GUI for upgrades: it's scriptable, it's more verbose about what's actually happening, and it doesn't leave you guessing whether a long-running job is progressing or hung.

This is a runbook, not a substitute for the version-specific release notes and upgrade/downgrade considerations — always read those for your exact target release first.

## Command reference

The operational commands, per Palo Alto's PAN-OS Upgrade Guide:

| Task | Command |
|---|---|
| Check current SW/content versions | `show system info` |
| Check available content updates (from PANW servers) | `request content upgrade check` |
| Show content versions known to the firewall | `request content upgrade info` |
| Download a content version | `request content upgrade download version <version>` |
| Install a content version | `request content upgrade install version <version>` |
| Check available software versions | `request system software info` |
| Check preferred releases only | `request system software info preferred` |
| Check base releases only | `request system software info base` |
| Refresh the available-software list from PANW | `request system software check` |
| Download a software version | `request system software download version <version>` |
| Check a download/job status | `show jobs id <jobid>` |
| Install a downloaded software version | `request system software install version <version>` |
| Reboot the firewall | `request restart system` |

Notes:

- **`request restart system`** is the confirmed reboot command.
- `show jobs id <jobid>` monitors any backgrounded job (download or install). `show jobs all` lists them.
- On an HA pair, append **`sync-to-peer yes`** to the content and software download/install commands to push the same image/content to the peer in one step.

## The upgrade path rule

You cannot skip feature releases. To move across multiple feature versions you step through each one, installing the **latest maintenance release** of each hop rather than its base image.

- Confirm the exact path for your target in PANW's "Determine the Upgrade Path" doc for that release.
- Rule of thumb for a manual upgrade: **install and boot from the latest maintenance release at each feature-version hop.** Only download a feature release's **base image** when the base is actually your target — otherwise you download the base to satisfy the path and install the maintenance release on top.
- Example (10.2 → 11.1): you go 10.2 → **11.0** → **11.1**, upgrading through 11.0 before you can install 11.1. On an HA pair, **both peers** must reach the same feature release before continuing to the next hop.

Also confirm **content is current before the software upgrade** — a new PAN-OS often requires a minimum Applications-and-Threats content version, and installing content first avoids a failed or blocked software install.

## Standalone firewall

Pre-flight (do these before touching software):

1. Confirm admin and management-network access, and confirm the upgrade path (above).
2. Read the target release's **release notes** and **upgrade/downgrade considerations**, and complete the **PAN-OS upgrade checklist**.
3. Back up: export the running config and generate a tech support file off-box.

```
# current state
show system info | match sw-version

# export running config to a TFTP/SCP server
tftp export configuration from running-config.xml to <server-ip>

# generate + export a tech support file (verbose; good pre-change snapshot)
tftp export tech-support to <server-ip>
```

Content first:

```
request content upgrade check
request content upgrade download version <content-version>
show jobs id <jobid>                 # wait for FIN
request content upgrade install version <content-version>
request content upgrade info         # confirm installed
```

Software:

```
request system software check                         # refresh the list
request system software info preferred                # find the preferred release

# download each hop along the path (base image, then target maintenance release)
request system software download version 11.0.0
show jobs id <jobid>                                  # wait for FIN
request system software download version <target 11.1 maintenance release>
show jobs id <jobid>

# install the target for this hop, then reboot into it
request system software install version <version>
# confirm 'y' when prompted; install runs in the background
show jobs id <jobid>                                  # wait for FIN

request restart system
```

After reboot:

```
show system info | match sw-version    # confirm the new version booted
show jobs all                          # confirm autocommit finished cleanly
```

If the path has more than one feature hop, repeat the software block for the next hop after the box is back up and healthy.

## Active/passive HA pair

The principle: **upgrade one peer at a time so the pair never loses service, and control which box is active at each step.** The order below fails over onto the secondary, upgrades the primary while it's passive, then repeats on the secondary.

### 1. Disable preemption (once, on the active box)

So the pair doesn't fail back mid-procedure the instant a box recovers.

```
configure
show deviceconfig high-availability group election-option    # note if preemptive is 'yes'
set deviceconfig high-availability group election-option preemptive no
commit
exit
```

### 2. Back up both peers

Export running config and a tech support file from **each** firewall (same commands as standalone). Do this on the active and the passive box.

### 3. Content + software to both, in one step each

From the active peer, using `sync-to-peer yes` so the passive gets the same bits:

```
request content upgrade download latest sync-to-peer yes
request content upgrade install sync-to-peer yes version latest

request system software download sync-to-peer yes version 11.0.0
request system software download sync-to-peer yes version <target 11.1 release>
```

Downloading only stages the images — nothing reboots yet.

### 4. Upgrade the primary (while it's passive)

Fail traffic onto the secondary, then install on the now-passive primary:

```
# on the ACTIVE (primary) box — hand off to the secondary
request high-availability state suspend
```

Confirm the **secondary is now active** and traffic still flows (log into the secondary; the prompt should no longer read passive). Then on the **primary** (now suspended/passive):

```
request system software install version <version>       # confirm 'y'
show jobs id <jobid>                                     # wait for FIN
request restart system
```

When the primary is back up:

```
show system info | match sw-version    # confirm new version
show high-availability state           # should return to functional (passive)
```

If it stays suspended after the reboot, un-suspend it: `request high-availability state functional`.

### 5. Upgrade the secondary

Now flip roles. Suspend the secondary, which makes the (upgraded) primary active again:

```
# on the SECONDARY box
request high-availability state suspend
```

Confirm the primary is active and traffic flows, then install + reboot on the secondary exactly as in step 4. When it's back:

```
show high-availability state           # both peers healthy, versions matched
```

Across a multi-hop path, both peers must land on the same feature release before you start the next hop — so you run steps 3–5 per hop (e.g. get both to 11.0, then both to 11.1).

### 6. Re-enable preemption (if you disabled it)

```
configure
set deviceconfig high-availability group election-option preemptive yes
commit
exit
```

## Verify

- `show system info | match sw-version` on each box — target version booted.
- `show high-availability state` and `show high-availability all` — pair healthy, config and session sync established, no split-brain.
- `request content upgrade info` — content at the expected version on both.
- `show jobs all` — the post-boot autocommit completed without errors.
- Spot-check real traffic and a failover *and* recovery before calling it done — recovery is the half that gets skipped and the half that generates tickets.

## Gotchas

- **Skipping feature releases fails.** You can't jump 10.2 → 11.1; step through 11.0. On HA, both peers reach each hop before the next.
- **Stale content blocks the software install.** Bring Applications-and-Threats content current before the PAN-OS upgrade.
- **Preemption left on** causes an unwanted fail-back mid-upgrade, right when the just-rebooted box may not be fully healthy. Disable it for the duration.
- **Don't close the CLI session during install.** Install runs in the background, but let it reach FIN (`show jobs id`) before rebooting.
- **`sync-to-peer yes` stages, it doesn't upgrade.** It only copies images/content to the peer; each box still installs and reboots on its own turn.
- **The newest content isn't always the top of the list** from `request content upgrade check` — read the versions, don't assume ordering.
- **Confirm plugin/PAN-OS compatibility** (SD-WAN, VM-Series, GlobalProtect, etc.) before upgrading either independently — see [gotchas.md](../gotchas/gotchas.md).

## References

- Use CLI Commands for Upgrade Tasks: [docs.paloaltonetworks.com/pan-os/11-1/pan-os-upgrade/cli-commands-for-upgrade/use-cli-commands-for-upgrade-tasks](https://docs.paloaltonetworks.com/pan-os/11-1/pan-os-upgrade/cli-commands-for-upgrade/use-cli-commands-for-upgrade-tasks)
- Determine the Upgrade Path to PAN-OS 11.1: [docs.paloaltonetworks.com/pan-os/11-1/pan-os-upgrade/upgrade-pan-os/upgrade-the-firewall-pan-os/determine-the-upgrade-path](https://docs.paloaltonetworks.com/pan-os/11-1/pan-os-upgrade/upgrade-pan-os/upgrade-the-firewall-pan-os/determine-the-upgrade-path)
- Upgrade an HA Firewall Pair: [docs.paloaltonetworks.com/pan-os/11-1/pan-os-upgrade/upgrade-pan-os/upgrade-the-firewall-pan-os/upgrade-an-ha-firewall-pair](https://docs.paloaltonetworks.com/pan-os/11-1/pan-os-upgrade/upgrade-pan-os/upgrade-the-firewall-pan-os/upgrade-an-ha-firewall-pair)
- PAN-OS 11.1 Release Notes: [docs.paloaltonetworks.com/pan-os/11-1/pan-os-release-notes](https://docs.paloaltonetworks.com/pan-os/11-1/pan-os-release-notes)
