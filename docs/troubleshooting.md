# Troubleshooting

Problem, cause, fix. Add each issue you hit during the lab.

## 1. SSH login with a key file that does not exist

**Problem:** `ssh -i ~/.ssh/lab_key_linux root@LAB_IP` printed `Identity file ... not accessible: No such file or directory`, but the login still worked.

**Cause:** the key file is `~/.ssh/lab_key`. `lab-server-linux` is only the key's comment (set with `-C`), not its file name. When the `-i` file is missing, SSH warns and falls back to the keys loaded in ssh-agent. The agent happened to hold `lab_key`, so the login worked by luck.

**Fix:** use the real file and stop SSH from offering other keys:
```
ssh -i ~/.ssh/lab_key -o IdentitiesOnly=yes root@LAB_IP
```

## 2. scp result saved under the wrong name

**Problem:** `results/before/lynis.txt` was missing; the file had been saved as `lynix.txt`. `ports.txt` was not copied at all on the first try.

**Fix:** renamed the file, re-ran the second `scp`, then checked with `ls -la results/before/`.

The lynis output also contained terminal colour codes (`[0;36m`). Removed with:
```
sed -i -E 's/\x1b\[[0-9;]*[A-Za-z]//g' results/before/lynis.txt
```

## 3. nmap shows ports the server does not have (ISP interception)

**Problem:** the "before" nmap from the home network showed five ports:

| Port | nmap result |
|---|---|
| 22/tcp | open, OpenSSH 9.6p1 Ubuntu 3ubuntu13.18 |
| 53/tcp | open, ISC BIND |
| 2323/tcp | open, OpenSSH 10.2p1 Ubuntu 2ubuntu3.8 |
| 6969/tcp | filtered |
| 7512/tcp | open, Cisco IOS telnetd |

On the server itself, `ss -tulpn` showed only port 22 listening publicly. DNS (53) listens on localhost only, and nothing listens on 2323, 6969 or 7512. The SSH version on 2323 is also different from the server's.

**Investigation** (all from the local PC, nothing changed on the server):

1. **Banner grab:** connecting to `LAB_IP:22` returned the server's real banner (`OpenSSH_9.6p1 Ubuntu-3ubuntu13.18`). Port 2323 returned `OpenSSH_10.2p1`, and 7512 returned a Cisco `User Access Verification / Username:` prompt.
2. **Reserved-address test:** `192.0.2.1` is a reserved documentation address (TEST-NET-1) that exists nowhere on the internet. Connecting to `192.0.2.1:2323` and `192.0.2.1:7512` still returned the **same** SSH and Cisco answers. Port 22 on `192.0.2.1` got no answer. So something on the path answers ports 2323 and 7512 for **any** destination.
3. **Local PC ruled out:** no listener on 2323 or 7512 (`ss -tlnp`), no VPN or proxy service, and no redirect rules:
   ```
   sudo nft list ruleset | grep -inE '2323|7512|dnat|redirect'   # no matches
   ```
4. **Home router ruled out:** on the TP-Link Archer AX55, port forwarding and port triggering had no rules. DMZ, UPnP and the VPN client were off. HomeShield was not subscribed, and the ALGs were the standard ones. The router's WAN address and gateway are private addresses, which means the ISP uses carrier-grade NAT (CGNAT).
5. **Hop test:** connected to `192.0.2.1:2323` and `:7512` with the IP TTL limited to 1, 2, 3 ... (no root needed):

   | TTL | Hop (from `tracepath -n 192.0.2.1`) | Connection |
   |---|---|---|
   | 1 | home router | no answer |
   | 2 | ISP gateway (10.x) | no answer |
   | 3 | ISP internal network (10.x) | **connected** |

**Cause:** equipment inside the ISP's network, 3 hops out, intercepts every outgoing connection to ports 2323 and 7512 and answers itself. These are classic IoT-malware (Mirai-style) telnet/SSH ports, so this is most likely an ISP honeypot or sinkhole. Only the ISP could confirm the exact reason.

**Fix / how the results are read:**
- Ports 53, 2323, 6969 and 7512 in the nmap output are artefacts of the home network, not of the server. Port 22 is real.
- The raw nmap output is kept unchanged in `results/before/nmap.txt`. Nothing was edited out.
- The "after" scan runs from the same network, so the same artefacts are expected there too.

**Lesson:** never trust a scan result alone. Compare it with what the server itself reports (`ss -tulpn`) and test a reserved address to see if the network path is answering instead of the target.
