# Lab runbook

Every command, in order. Real IP written as LAB_IP.

## Step 1: create the server and measure "before"

### 1.0 SSH key (local machine)
```
ssh-keygen -t ed25519 -f ~/.ssh/lab_key -C "lab-server-linux"
cat ~/.ssh/lab_key.pub
```
Pasted the public key into Hetzner when creating the server.

### 1.1 Create the server (Hetzner console)
- Type: CX23 (Cost-Optimized, x86, 2 vCPU, 4 GB RAM, 40 GB), €0.010/h
- Location: Helsinki
- Image: Ubuntu 24.04
- Networking: IPv4 + IPv6
- SSH key: `lab_key`
- Renamed the server in the console to `lab`

Problem: the form defaulted to Ubuntu 26.04 and once to a CCX13 dedicated plan (€43.42/mo). Fix: picked the CX23 row and Ubuntu 24.04 by hand and checked the summary before creating.

### 1.2 First login (local machine)
```
ssh -i ~/.ssh/lab_key -o IdentitiesOnly=yes root@LAB_IP
```
Problem: first tried `-i ~/.ssh/lab_key_linux`, a file that does not exist. SSH fell back to the keys in ssh-agent and got in by luck. Fix: the file is `~/.ssh/lab_key`; `lab-server-linux` is only the key's comment. `IdentitiesOnly=yes` stops SSH offering other keys.

### 1.3 Check the server (lab server)
```
cat /root/.ssh/authorized_keys
cat /etc/os-release | head -3
hostnamectl set-hostname lab
exit
```
Result: one authorized key (matches `lab_key.pub`), Ubuntu 24.04.4 LTS, hostname set to `lab`.

### 1.4 Measure "before" (lab server, as root)
```
apt update
apt install -y lynis
lynis audit system --quick | tee /root/lynis-before.txt
ss -tulpn | tee /root/ports-before.txt
grep -i "hardening index" /root/lynis-before.txt
exit
```
No `apt upgrade`, so the server stays as delivered. Ignored the login banner's offer to upgrade to 26.04.

Result: **Hardening index 58**, 259 tests, 3 warnings, 46 suggestions. Only SSH (22) listens publicly (IPv4 and IPv6). DNS (53) is localhost only.

### 1.5 External scan (local machine, lab server only)
```
nmap -sV LAB_IP | tee results/before/nmap.txt
```
Result: 22/tcp open, OpenSSH 9.6p1. nmap also showed 53, 2323, 6969 and 7512, which are not on the server. Proven to be ISP interception: see `troubleshooting.md` #3.

### 1.6 Copy results off the server (local machine)
```
scp -i ~/.ssh/lab_key -o IdentitiesOnly=yes root@LAB_IP:/root/lynis-before.txt results/before/lynis.txt
scp -i ~/.ssh/lab_key -o IdentitiesOnly=yes root@LAB_IP:/root/ports-before.txt results/before/ports.txt
sed -i -E 's/\x1b\[[0-9;]*[A-Za-z]//g' results/before/lynis.txt
```
Then replaced the server's IPv4, IPv6 and reverse-DNS name with `LAB_IP`, `LAB_IPV6` and `LAB_HOSTNAME`, and checked with:
```
grep -rE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' .
```
Problem: lynis file first saved as `lynix.txt`, and `ports.txt` not copied. See `troubleshooting.md` #2.

## Step 2: harden the server

### 2.1 Create a sudo user (lab server, as root)
```
adduser sakib
usermod -aG sudo sakib
mkdir -p /home/sakib/.ssh
cp /root/.ssh/authorized_keys /home/sakib/.ssh/authorized_keys
chown -R sakib:sakib /home/sakib/.ssh
chmod 700 /home/sakib/.ssh
chmod 600 /home/sakib/.ssh/authorized_keys
```
Tested in a second terminal with the root session still open:
```
ssh -i ~/.ssh/lab_key -o IdentitiesOnly=yes sakib@LAB_IP
sudo whoami    # prints: root
```
Result: key login as `sakib` works, and sudo works. The password is new and random, used only here.

### 2.2 Lock down SSH (lab server, as sakib; root session kept open)
Wrote `/etc/ssh/sshd_config.d/00-hardening.conf` (copy in `configs/ssh/00-hardening.conf`):
```
sudo tee /etc/ssh/sshd_config.d/00-hardening.conf > /dev/null <<'EOF'
... (see configs/ssh/00-hardening.conf)
EOF
sudo sshd -t
sudo sshd -T | grep -E '^(permitrootlogin|passwordauthentication|kbdinteractiveauthentication|allowusers|maxauthtries) '
sudo systemctl restart ssh
```
Tested in a new terminal before closing the root session:
```
ssh -i ~/.ssh/lab_key -o IdentitiesOnly=yes sakib@LAB_IP   # works
ssh -i ~/.ssh/lab_key -o IdentitiesOnly=yes root@LAB_IP    # Permission denied
```
Result: root login and password login disabled, only `sakib` with a key can log in.
