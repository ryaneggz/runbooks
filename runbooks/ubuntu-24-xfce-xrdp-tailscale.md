# Ubuntu 24.04 remote dev VM: XFCE + XRDP + Tailscale

- **Trigger:** Provision or rebuild an Ubuntu 24.04 VM that needs a lightweight remote GUI, with SSH and RDP inaccessible from the public network.
- **Owner:** ryaneggz
- **Severity:** SEV3
- **Last reviewed:** 2026-09-17

## Target state

- Ubuntu 24.04 LTS
- XFCE desktop for low-overhead remote sessions
- XRDP using its Xorg backend
- Tailscale for private connectivity
- SSH (`22/tcp`) and RDP (`3389/tcp`) reachable through the Tailscale path, not the VM's public interface
- UFW default-deny for unsolicited public inbound traffic
- No GNOME desktop required

> This runbook keeps SSH and RDP off the public network. Tailscale itself installs netfilter rules for tailnet traffic. If the requirement is to restrict *tailnet members* to only ports 22 and 3389, enforce that separately with Tailscale access-control policy.

## Prerequisites

- Fresh Ubuntu 24.04 LTS VM.
- A sudo-capable Linux user.
- Temporary public SSH or provider-console access for bootstrap.
- Tailscale installed and authenticated on the client machine that will administer the VM.
- Access to the provider firewall/security-group configuration, if one exists.

## Steps

1. Confirm the VM is Ubuntu 24.04.

   ```bash
   . /etc/os-release
   echo "$PRETTY_NAME"
   ```

   Expected: `Ubuntu 24.04... LTS`.

2. Update the base system.

   ```bash
   sudo apt update
   sudo apt upgrade -y
   ```

   If APT reports that `/var/lib/dpkg/lock-frontend` is held by `unattended-upgr`, let `unattended-upgrade` finish. Do not delete the lock file or kill an active package transaction.

   Check whether a reboot is required:

   ```bash
   if [ -f /var/run/reboot-required ]; then cat /var/run/reboot-required; fi
   ```

   If required, reboot and reconnect:

   ```bash
   sudo reboot
   ```

3. Install XFCE and XRDP.

   ```bash
   sudo apt install -y xfce4 xfce4-goodies xrdp
   ```

   Verify the XRDP Xorg backend and XFCE session packages are present:

   ```bash
   dpkg -l | grep -E '^ii\s+(xrdp|xorgxrdp|xfce4|xfce4-session)\b'
   ```

   Expected: `xrdp`, `xorgxrdp`, `xfce4`, and `xfce4-session` are installed.

   If `xorgxrdp` is missing:

   ```bash
   sudo apt install -y xorgxrdp
   ```

4. Configure XRDP sessions for this user to start XFCE.

   ```bash
   printf '%s\n' 'xfce4-session' > ~/.xsession
   cat ~/.xsession
   ```

   Expected:

   ```text
   xfce4-session
   ```

5. Ensure the Linux user has a password XRDP can authenticate.

   If the account normally uses SSH keys and does not have a known local password, set one:

   ```bash
   sudo passwd "$USER"
   ```

   XRDP will use this Linux username and password.

6. Enable XRDP.

   ```bash
   sudo adduser xrdp ssl-cert
   sudo systemctl enable --now xrdp
   ```

   Verify:

   ```bash
   systemctl is-active xrdp
   sudo ss -lntp | grep ':3389'
   ```

   Expected: `xrdp` is `active` and TCP 3389 is listening.

7. Install and authenticate Tailscale on the VM.

   ```bash
   curl -fsSL https://tailscale.com/install.sh | sh
   sudo systemctl enable --now tailscaled
   sudo tailscale up
   ```

   Open the authentication URL printed by `tailscale up` and authorize the VM.

   Verify:

   ```bash
   tailscale status
   tailscale ip -4
   ```

   Record the `100.x.y.z` address as `<TAILSCALE_IP>`.

8. Prove SSH works through Tailscale **before enabling the firewall**.

   From a second terminal on the client machine:

   ```bash
   ssh <USER>@<TAILSCALE_IP>
   ```

   Expected: the SSH login succeeds.

   Keep this Tailscale SSH session open while changing UFW. Do not continue if Tailscale SSH does not work.

9. Configure UFW so SSH and RDP are not publicly reachable.

   ```bash
   sudo ufw default deny incoming
   sudo ufw default allow outgoing
   sudo ufw allow in on tailscale0 to any port 22 proto tcp
   sudo ufw allow in on tailscale0 to any port 3389 proto tcp
   ```

   Inspect existing rules:

   ```bash
   sudo ufw status numbered
   ```

   Remove generic public SSH/RDP allows if they exist:

   ```bash
   sudo ufw delete allow 22/tcp
   sudo ufw delete allow 3389/tcp
   ```

   `Could not delete non-existent rule` is harmless if the rule was never present.

   For better odds of a direct Tailscale path instead of DERP relay, optionally permit Tailscale's default WireGuard UDP port:

   ```bash
   sudo ufw allow 41641/udp
   ```

   This exposes Tailscale transport only; it does not expose SSH or RDP publicly. If the Tailscale UDP listen port has been customized, use that port instead.

   Enable and reload UFW:

   ```bash
   sudo ufw enable
   sudo ufw reload
   sudo ufw status verbose
   ```

   Expected: default incoming policy is deny, and there are no generic public allow rules for TCP 22 or TCP 3389.

10. Remove public SSH/RDP rules from the provider firewall.

    If the VPS/cloud provider has a firewall or security group, remove public inbound rules for:

    ```text
    TCP 22
    TCP 3389
    ```

    If using the optional direct Tailscale path, UDP 41641 may remain allowed.

11. Verify private connectivity from the client.

    ```bash
    nc -vz <TAILSCALE_IP> 22
    nc -vz <TAILSCALE_IP> 3389
    tailscale ping <TAILSCALE_IP>
    ```

    Expected: SSH and RDP succeed through the Tailscale IP. Prefer `tailscale ping` showing a direct endpoint rather than `via DERP(...)` for best interactive RDP performance.

12. Verify the public IP no longer exposes SSH or RDP.

    From outside the VM:

    ```bash
    nc -vz <PUBLIC_IP> 22
    nc -vz <PUBLIC_IP> 3389
    ```

    Expected: both fail or time out.

13. Connect over RDP using the Tailscale IP.

    From Windows or WSL:

    ```bash
    /mnt/c/Windows/System32/mstsc.exe /v:<TAILSCALE_IP>
    ```

    On macOS, add `<TAILSCALE_IP>` as the PC in an RDP client.

    At the XRDP login screen use:

    ```text
    Session: Xorg
    Username: <Linux username>
    Password: <Linux password>
    ```

    Expected: the session opens into XFCE.

## Verification

Run on the VM:

```bash
echo '=== OS ==='
. /etc/os-release && echo "$PRETTY_NAME"

echo '=== XRDP ==='
systemctl is-active xrdp
sudo ss -lntp | grep ':3389'

echo '=== XFCE session ==='
cat ~/.xsession

echo '=== Tailscale ==='
systemctl is-active tailscaled
tailscale ip -4

echo '=== Firewall ==='
sudo ufw status verbose
```

Expected:

- Ubuntu 24.04 LTS.
- `xrdp` is active.
- `~/.xsession` contains `xfce4-session`.
- `tailscaled` is active and has a `100.x.y.z` address.
- UFW is active with default incoming deny.
- No generic public `22/tcp` or `3389/tcp` allow rule exists.

From the client:

```bash
nc -vz <TAILSCALE_IP> 22
nc -vz <TAILSCALE_IP> 3389
tailscale ping <TAILSCALE_IP>
```

## Performance notes

XFCE is intentional. On a headless VPS, GNOME over XRDP can feel sluggish even when CPU, RAM, disk, and RDP resolution are not constrained. XFCE reduces graphical/session overhead.

If RDP still feels slow:

```bash
tailscale ping <TAILSCALE_IP>
```

If the path stays on `DERP`, permit the VM's Tailscale UDP listening port (41641 by default) in UFW and the provider firewall, then retest.

Check session resolution from inside RDP:

```bash
xrandr | grep '\*'
```

## Migrating an existing GNOME/XRDP VM to XFCE

If `ubuntu-desktop-minimal` was installed before XFCE, first switch XRDP to XFCE:

```bash
sudo apt install -y xfce4 xfce4-goodies
echo 'xfce4-session' > ~/.xsession
sudo systemctl restart xrdp
```

After confirming XFCE works, optional GNOME cleanup is:

```bash
sudo systemctl disable --now gdm3
sudo systemctl set-default multi-user.target
sudo apt purge -y ubuntu-desktop-minimal
sudo apt autoremove --purge --dry-run
```

Review the dry-run carefully. Do not remove `xrdp`, `xorgxrdp`, `xfce4`, `xfce4-session`, `openssh-server`, or `tailscale`.

If the preview is correct:

```bash
sudo apt autoremove --purge -y
sudo apt clean
```

## Rollback

### Disable RDP but keep SSH over Tailscale

```bash
sudo systemctl disable --now xrdp
sudo ufw delete allow in on tailscale0 to any port 3389 proto tcp
sudo ufw reload
```

### Remove the GUI and XRDP

```bash
rm -f ~/.xsession
sudo apt purge -y xfce4 xfce4-goodies xrdp xorgxrdp
sudo apt autoremove --purge -y
sudo apt clean
```

### Restore public SSH only if required

Do this only when deliberately abandoning Tailscale-only administration. Add the provider firewall rule and UFW rule **before** disconnecting the working Tailscale SSH session:

```bash
sudo ufw allow 22/tcp
```

Verify a new public-IP SSH session works before changing or removing Tailscale.

## Troubleshooting / escalation

If XRDP accepts credentials and immediately disconnects, collect:

```bash
sudo tail -n 100 /var/log/xrdp-sesman.log
sudo tail -n 100 /var/log/xrdp.log
journalctl -u xrdp -b --no-pager | tail -100
cat ~/.xsession
```

If Tailscale connectivity is slow or unavailable, collect:

```bash
tailscale status
tailscale ping <CLIENT_OR_VM_TAILSCALE_IP>
sudo ufw status verbose
```

Do not start modifying `/etc/xrdp/startwm.sh` unless the stock XFCE session path above has been verified and the logs identify a session-start problem.

## References

- Tailscale: https://tailscale.com/docs/how-to/secure-ubuntu-server-with-ufw
- Tailscale firewall ports: https://tailscale.com/docs/reference/faq/firewall-ports
- Ubuntu 24.04 `xrdp`: https://packages.ubuntu.com/noble/xrdp
- Ubuntu 24.04 `xorgxrdp`: https://packages.ubuntu.com/noble/xorgxrdp
