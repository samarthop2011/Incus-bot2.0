# ◆ Incus VPS Hosting Bot (Local Node Edition)

Invite-based VPS deployment bot for a **single local Incus host**. No external nodes, no SSH
between machines, and **no public IPv4/IPv6 required**.

## How it works without a public IP
- The bot runs on the same machine as Incus and uses the local `incus` CLI (no shell, no SSH).
- VPSes live on a private NAT bridge (`incusbr0`) with outbound internet only.
- Users get access through **tmate** or **sshx** tunnels, which are outbound connections.
- The Incus API is **not** exposed. Only the local unix socket is used.

## Requirements
- Ubuntu 22.04/24.04 or Debian 12/13, root access, Python 3.10+
- Outbound internet on the host (Discord, images.linuxcontainers.org, tmate.io, sshx.io, github.com)

## Quick start
```bash
# 1) Install + initialise Incus (btrfs pool for disk quotas, NAT bridge, self-test)
sudo POOL_SIZE_GB=200 bash scripts/setup_host.sh

# 2) Bot environment
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env && nano .env      # DISCORD_TOKEN, GUILD_ID
nano config.json                       # channel IDs, role IDs, capacity

# 3) Run
python3 bot.py

# 4) (optional) run 24/7
sudo bash scripts/install_service.sh
```

In Discord, run `/panel` (or `!panel`) as an admin to post the permanent deploy panel.

Discord Developer Portal: enable **Server Members Intent** and **Message Content Intent**.
Bot permissions: Manage Server (invite tracking), Send Messages, Embed Links, Manage Roles.

## Config reference (`config.json` → `host`)
| Key | Meaning |
| :--- | :--- |
| `max_total_vps` | Hard cap on VPS count on this machine |
| `ram_overcommit` | Allocatable RAM = (host RAM - reserve) x this |
| `cpu_overcommit` | Allocatable vCPU = host threads x this |
| `reserve_ram_mb` | RAM kept free for the host/OS |
| `min_free_disk_gb` | Refuse deploys if host free disk is below this |
| `max_concurrent_deploys` | Parallel deployments (others are queued) |
| `max_snapshots_per_vps` | Oldest snapshot is auto-deleted past this |
| `nesting` | Enables `security.nesting` (Docker inside VPS). Reduces isolation slightly |
| `allow_unquoted_disk` | Allow deploys on pools without quota support (dir). Not recommended |

Check image names exist: `incus image list images: debian 12`. Edit `images` in `config.json` as needed.

## Commands (all work as `/slash` and `!prefix`)
User: `deploy` `plans` `myvps` `renew` `invites` `leaderboard` `whoinvited` `status` `help` `ping` `botinfo`
Admin: `panel`, `admin stats | host | vps | info | extend | addinv | blacklist | unblacklist | suspend | unsuspend | terminate | forcevps | maintenance | broadcast | purge`

## Troubleshooting
**Container has no internet / never gets an IPv4**
- UFW: `sudo ufw allow in on incusbr0 && sudo ufw route allow in on incusbr0 && sudo ufw route allow out on incusbr0`
- Docker on the host sets FORWARD to DROP: 
  `sudo iptables -I DOCKER-USER -i incusbr0 -j ACCEPT`
  `sudo iptables -I DOCKER-USER -o incusbr0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT`
- Check: `incus network show incusbr0` and `incus list`

**"Failed to apply quota"**: your pool is `dir`. Use btrfs/zfs (setup script default) or LVM.

**Bot not root?** Add its user to the `incus-admin` group, or run as root.

## Security notes
- Containers on the same bridge can reach each other's private IPs. No SSH server is opened
  by default, which limits exposure. For strict tenant isolation look into Incus network ACLs.
- Root passwords are shown once and not stored. Users can reset them from `/myvps`.
- tmate/sshx links give a root shell to anyone holding them. Treat them like passwords.
- Reaching VPS services from outside needs your own tunnel, such as cloudflared or Tailscale
  installed inside the VPS. The bot does not open any inbound ports.
