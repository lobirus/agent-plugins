---
name: new-server
description: Bootstrap a fresh Ubuntu/Debian server the owner's way — byobu, a passwordless-sudo admin user with key-only SSH on a custom port, root locked, ufw, UTC + NTP, full upgrade and reboot, a /data disk, Docker, and a Git SSH key for deploying apps from repos.
argument-hint: "[root@]<host-or-ip>"
disable-model-invocation: true
---

# New server setup

Target: $ARGUMENTS

You run on the user's local machine and drive the server over `ssh` from your shell tool. Work through the phases in order. Each phase starts by checking whether its work is already done, so a rerun after a failure resumes instead of repeating it.

## Ground rules

- **Never lock the user out.** Don't disable a way in until its replacement has been tested in a *fresh* SSH connection. If a check fails, stop, show the output, and fix it before moving on.
- Every ssh call is non-interactive: `ssh -o BatchMode=yes -o ConnectTimeout=10 -o StrictHostKeyChecking=accept-new ...`. On a host key **mismatch** error, stop and ask the user. The server may have been reinstalled; offer `ssh-keygen -R <host>` (and `-R "[<host>]:<port>"`).
- Run remote work as a script: `ssh ... 'sudo bash -s' <<'EOF'` with `set -euo pipefail` first, and write concrete values (username, port, device) literally into the script text.
- Prefix every apt command with `DEBIAN_FRONTEND=noninteractive`.
- Ask the user for decisions with your question tool if you have one (AskUserQuestion in Claude Code), and batch related questions into one ask.
- Supported: Ubuntu 22.04+ and Debian 12+. On anything else, stop and tell the user.
- Keep track of HOST, NEW_USER, SSH_PORT, ALIAS, LOCAL_KEY, DATA_DEVICE; the summary at the end prints them.

## Phase 0 — Preflight (local)

1. Take HOST from the arguments, without any `root@`. If it's empty, ask.
2. Find the local public key with `ls ~/.ssh/*.pub`. Prefer `id_ed25519.pub`, then `id_rsa.pub`; if there are several others, ask which one. If there are none, ask before creating one with `ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519`. Tell the user it will have no passphrase unless they run the command themselves.
3. Test root access: `ssh <opts> root@HOST 'cat /etc/os-release; id -u'`.
   - `Permission denied` usually means the server only accepts a password. Ask the user to run `ssh-copy-id -i <LOCAL_KEY> root@HOST` in their own terminal, where they can type the password, then test again.
   - If the provider gave a sudo user instead of root (such as `ubuntu`), connect as that user and prefix commands with `sudo` wherever this skill says root.
4. Check the OS against the supported list.

## Phase 1 — Questions

Generate a random port locally with `echo $((20000 + RANDOM % 40000))`, then ask in one question:

- **Username** for the new admin user. Suggest the local `whoami`.
- **SSH port**: the random port (Recommended), or Other.

Check on the server that the port is free: `ss -tlnH "sport = :PORT"` must print nothing.

## Phase 2 — Base packages (as root)

```bash
apt-get update
apt-get install -y sudo ufw byobu chrony curl ca-certificates gnupg
```

## Phase 3 — Admin user (as root)

```bash
set -euo pipefail
U=NEW_USER
id "$U" >/dev/null 2>&1 || useradd -m -s /bin/bash "$U"
usermod -aG sudo "$U"
usermod -p '*' "$U"   # no password, but not "locked" ('!'), so key logins work under any sshd UsePAM setting
echo "$U ALL=(ALL) NOPASSWD:ALL" > "/etc/sudoers.d/90-$U"
chmod 440 "/etc/sudoers.d/90-$U"
visudo -cf "/etc/sudoers.d/90-$U"
install -d -m 700 -o "$U" -g "$U" "/home/$U/.ssh"
touch "/home/$U/.ssh/authorized_keys"
chown "$U:$U" "/home/$U/.ssh/authorized_keys"; chmod 600 "/home/$U/.ssh/authorized_keys"
```

Copy the local key without making duplicates:

```bash
KEY="$(cat LOCAL_KEY)"
ssh <opts> root@HOST "grep -qxF '$KEY' /home/NEW_USER/.ssh/authorized_keys || echo '$KEY' >> /home/NEW_USER/.ssh/authorized_keys"
```

Turn on byobu for the user with `sudo -u NEW_USER -H byobu-enable`. Use `-H`, not `-i`: after this, a login shell starts byobu.

**Check (fresh connection):** `ssh <opts> NEW_USER@HOST 'sudo -n true && echo OK'` must print `OK`.
From now on, connect only as NEW_USER and use `sudo`. Don't use root again.

## Phase 4 — Time: UTC + NTP

```bash
sudo timedatectl set-timezone Etc/UTC
sudo systemctl enable --now chrony
```

Check that `timedatectl` shows `Time zone: Etc/UTC`, and look at `chronyc tracking`. Syncing can take a minute, so don't block on it; check again after the reboot.

## Phase 5 — Firewall, first pass

Keep port 22 open for now, so the current way in still works:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp comment 'ssh (temporary)'
sudo ufw allow SSH_PORT/tcp comment 'ssh'
sudo ufw --force enable
```

Don't open 123/udp (NTP). chrony syncs over outgoing traffic, and ufw lets the replies back in; the port only needs to be open for a server that hands out time to other machines.

**Check:** a fresh `ssh <opts> NEW_USER@HOST true` still works on port 22.

## Phase 6 — SSH hardening, listening on both ports first

1. Make sure `/etc/ssh/sshd_config` starts with `Include /etc/ssh/sshd_config.d/*.conf`; add it at the top if it's missing. Comment out any active `Port` line in the main file, because `Port` values add up across files.
2. Write `/etc/ssh/sshd_config.d/00-hardening.conf`. The `00-` prefix matters: sshd keeps the first value it reads, so this file wins over cloud-init's `50-cloud-init.conf`, which often turns passwords back on.
   ```
   Port 22
   Port SSH_PORT
   PermitRootLogin no
   PasswordAuthentication no
   KbdInteractiveAuthentication no
   PubkeyAuthentication yes
   ```
3. `sudo sshd -t`, and stop if it fails. Then apply the change. Newer Ubuntu uses socket activation, where the port comes from the socket unit:
   ```bash
   if systemctl is-enabled --quiet ssh.socket 2>/dev/null; then
     sudo systemctl daemon-reload && sudo systemctl restart ssh.socket
   fi
   sudo systemctl restart ssh
   ```
4. Confirm the settings took effect: `sudo sshd -T | grep -Ei '^(port|permitrootlogin|passwordauthentication|kbdinteractiveauthentication) '`.
5. **Check, each in a fresh connection from local:**
   - `ssh <opts> -p SSH_PORT NEW_USER@HOST 'sudo -n true && echo OK'` prints `OK`.
   - `ssh <opts> -p SSH_PORT root@HOST true` **fails** with `Permission denied`.
   - `ssh -o BatchMode=no -o PubkeyAuthentication=no -o PreferredAuthentications=password,keyboard-interactive -o NumberOfPasswordPrompts=0 -p SSH_PORT NEW_USER@HOST true` **fails** with `Permission denied (publickey)`.
6. Only when all three pass, finish the switch:
   - Remove the `Port 22` line from `00-hardening.conf`, then repeat steps 3 and 4.
   - `sudo ufw delete allow 22/tcp`
   - Check the new port again with the first test from step 5, and check that port 22 now refuses or times out.
7. Lock root's password with `sudo passwd -l root`. Tell the user that root can't be deleted, because the system runs as root, but it's now locked and can't log in over SSH.

Tell the user the new port clearly: **"SSH now listens on port SSH_PORT."**

## Phase 7 — ~/.ssh/config entry

Suggest the server's short hostname (`hostname -s`) as ALIAS. Show the block and ask before writing anything; offer Yes, Yes with a different name (Other), or No:

```
Host ALIAS
    HostName HOST
    User NEW_USER
    Port SSH_PORT
    IdentityFile LOCAL_KEY_WITHOUT_.pub
    IdentitiesOnly yes
```

If the user says yes:
- Check that no `Host ALIAS` exists yet; if one does, ask.
- Create `~/.ssh/config` with mode 600 if it's missing, then append the block.
- Check that `ssh ALIAS true` works.

From here on, use `ssh ALIAS` (or the full flags if the user said no).

## Phase 8 — Upgrade and reboot

```bash
sudo DEBIAN_FRONTEND=noninteractive apt-get update
sudo DEBIAN_FRONTEND=noninteractive apt-get -y -o Dpkg::Options::=--force-confdef -o Dpkg::Options::=--force-confold full-upgrade
sudo DEBIAN_FRONTEND=noninteractive apt-get -y autoremove --purge
```

Reboot, and use the boot ID so you know it really rebooted:

1. Save `cat /proc/sys/kernel/random/boot_id`.
2. Run `sudo systemctl reboot`. An ssh exit code of 255 is expected here.
3. Every 10 s, for up to 5 min, try to read `boot_id`. Continue when it reads back and differs from the saved one.

After the reboot, check that ssh works on SSH_PORT, `sudo ufw status verbose` allows only SSH_PORT/tcp, and `timedatectl` reports `System clock synchronized: yes`.

## Phase 9 — /data disk

Skip this phase if `findmnt /data` already shows a mount.

Look at the disks:
```bash
lsblk -J -b -o NAME,PATH,TYPE,SIZE,FSTYPE,MOUNTPOINTS,MODEL,SERIAL
findmnt -no SOURCE /
swapon --show
```

A disk is a **candidate** only if all of these hold:
- Its `TYPE` is `disk`, not loop, rom, or zram.
- It has no partitions, no `FSTYPE`, and no mountpoints.
- It isn't the root or swap disk.
- `sudo wipefs -n DEV` prints nothing.

List separately any disk that has a filesystem but is mounted somewhere else, for example a provider volume mounted at `/mnt/HC_Volume_*`. Ask whether the user wants that disk at /data instead.

**Ask before formatting. Never format without an explicit yes.** Name the device, size, and model, and say plainly that formatting erases it. If there are several candidates, ask which one to use.

Format and mount it:
```bash
set -euo pipefail
DEV=DATA_DEVICE
sudo mkfs.ext4 -L data "$DEV"
UUID=$(sudo blkid -s UUID -o value "$DEV")
sudo mkdir -p /data
grep -q "UUID=$UUID" /etc/fstab || echo "UUID=$UUID /data ext4 defaults,nofail 0 2" | sudo tee -a /etc/fstab
sudo findmnt --verify
sudo mount /data
```

If no disk is suitable, tell the user and create `/data` as a plain directory on the root filesystem.

Then, in either case:
```bash
sudo mkdir -p /data/apps
sudo chown NEW_USER:NEW_USER /data /data/apps
```

## Phase 10 — Docker

Set up Docker's official apt repository, with `$ID` and `$VERSION_CODENAME` read from `/etc/os-release`:
```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL "https://download.docker.com/linux/$ID/gpg" -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/$ID $VERSION_CODENAME stable" | sudo tee /etc/apt/sources.list.d/docker.list
```

Write `/etc/docker/daemon.json` **before** installing, so Docker never starts with the default settings. Include `data-root` only if /data is a separate disk:
```json
{
  "data-root": "/data/docker",
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" }
}
```

```bash
sudo DEBIAN_FRONTEND=noninteractive apt-get update
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker NEW_USER
```

**Check (fresh connection, so the new group applies):** `docker run --rm hello-world`, `docker compose version`, and `docker info --format '{{.DockerRootDir}}'`.

Warn the user: **ports that Docker publishes bypass ufw**, because Docker writes its own iptables rules. In compose, write `127.0.0.1:8080:80` for anything that should only be reached through a reverse proxy, and publish on all interfaces only what should be public.

## Phase 11 — Git SSH key for deploying apps

Ask which Git host to use: GitHub (Recommended), GitLab, Bitbucket, or Other.

Run as NEW_USER, with no sudo:
```bash
test -f ~/.ssh/id_ed25519 || ssh-keygen -t ed25519 -N "" -C "NEW_USER@$(hostname -s)" -f ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub
```

Show the public key in a code block and say where to add it:
- **GitHub:** Settings → SSH and GPG keys → New SSH key. This gives access to every repo the account can reach. For a single repo, use that repo's Settings → Deploy keys instead; GitHub allows a deploy key on only one repo.
- **GitLab:** Preferences → SSH Keys, or the project's Settings → Repository → Deploy keys.

Ask the user to confirm with "I've added it" or "Skip for now". Then test with `ssh -T -o StrictHostKeyChecking=accept-new git@github.com` (or the GitLab or Bitbucket host). GitHub exits with code 1 even on success; treat `successfully authenticated` (GitLab: `Welcome to GitLab`) as a pass.

## Phase 12 — Summary

Print a short table: host, alias, user, SSH port, how to connect (`ssh ALIAS`), firewall rules (`sudo ufw status numbered`), time zone and NTP status, /data (device, size, filesystem, or "root filesystem"), the Docker version and its data directory, and the Git key's fingerprint.

Then give the next steps:
```bash
ssh ALIAS
cd /data/apps && git clone git@github.com:<owner>/<repo>.git && cd <repo>
docker compose up -d
```

Close with two reminders:
- NEW_USER has no password, so the provider's web console can't log in as NEW_USER. For emergency console access, set one with `sudo passwd NEW_USER`; SSH password login stays off.
- Ports that Docker publishes bypass ufw.
