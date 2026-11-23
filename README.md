# CCDC Invitational 3 Report

November 22, 2026

## 1. Overview

The Collegiate Cyber Defense Competition (CCDC) is a defense focused cybersecurity competition. Every team inherits the same network of vulnerable machines and has to defend it while keeping it running.

How the day works:

- **Uptime scoring.** A scoring engine checks each service every few minutes. If a service is up and working, the team earns points. If it is down, the team loses them.
- **Injects.** These are business tasks sent out in an email format that the team completes during the event as if working for a company.
- **Red Team.** A group of attackers goes after the network the whole time.
- **Objective.** Keep services alive and hardened under pressure, and communicate well as a team.

**Event:** WCComps 2026 Invitational 3 (November 21, 2026)

**Team:** Rutgers cyber club (RUSEC) Team 11

**Scenario:** Flokid, a fictional Icelandic furniture company. 10 VMs on 192.168.220.0/24 behind 1:1 NAT at 10.100.111.X.

**My role.** I was responsible for kvigindis (192.168.220.44), an Ubuntu 24.04 server running a furniture help website (PHP app in Docker behind Nginx). I also helped with the Wazuh SIEM on another box called gufu (192.168.220.100).

## 2. Tools and Network

| Tool | What it did |
|------|-------------|
| Docker | Ran the furniture help site as a containerized PHP app |
| Nginx | Reverse proxy in front of the Docker container |
| PHP | Backend language for the furniture help site |
| UFW | Firewall on kvigindis |
| Wazuh | SIEM agent sending logs to the manager on gufu |
| Step CA | SSH certificate authority for certificate based login |
| Discord | Team communication |

## 3. My Experience

This was my third CCDC event and the growth from each one was clear. In Invitational 1, I was brand new and spent most of the day learning alongside a teammate on the logging box. In Invitational 2, I had more confidence but was still getting comfortable with the pace. By Invitational 3, I was fully responsible for a Linux box on my own and had to find, clean, and harden everything myself.

When I connected to kvigindis, I changed the default root credentials right away, then noticed the furniture help site was down. The database credentials in the app were out of sync with the actual MySQL database, so the site couldn't connect. I coordinated with the teammate responsible for the database server to get the MySQL user credentials synced up, and once that was done the site came back online. I also identified the red team IP (10.194.194.85) brute forcing in auth.log and told the router team to block it at the perimeter. I found that the SSH config had a critical misconfiguration. The line `AuthorizedPrincipalsFile none` meant the server would accept any SSH certificate as any user. Red team had already exploited this to get root access.

With the service up, I started taking inventory: checking users, listening ports, running services, and firewall state. I found that the SSH config had a critical misconfiguration that had given red team root access.

Once I realized red team had been in, I started hunting for persistence. Persistence is how attackers stay on a machine even after passwords are changed or sessions are killed. I found four separate backdoors they had planted:

1. A fake service account called `svc_health` with NOPASSWD sudo access, made to look like a legit monitoring user.
2. A systemd timer called `health-check.timer` with a matching service that ran a malicious script on a loop, calling back to the red team IP (10.194.194.85).
3. Malicious lines injected into root's `.bashrc` that set up a reverse shell every time someone opened a root terminal.
4. A hidden SSH authorized_keys file stashed in `/etc/ssh/.ssh/` (a hidden directory red team created) for passwordless access. Throughout the competition, red team was also brute forcing passwords from their IP (10.194.194.85), which I could see in auth.log.

I cleaned all four, then moved to hardening. I patched the SSH config by changing `AuthorizedPrincipalsFile none` to `AuthorizedPrincipalsFile /etc/ssh/auth_principals/%u` and created per user principal files so only certificates with the correct principal could log in as each user. I rotated the root password, set up UFW with only the ports I needed (22, 80, 443, 9000), and configured the Wazuh SIEM agent on kvigindis to report to the manager on gufu. I also verified the Docker container's PHP config had dangerous functions (exec, system, passthru, shell_exec) disabled to prevent PHP2Shell attacks. I SSHed into gufu to check on the Wazuh manager, but the VMs did not have enough resources to do much with it, so I decided to just focus on keeping my SLA uptime on kvigindis. I also helped a teammate who had an SSHole vulnerability on their Alpine machine by telling them to update their OpenSSH packages. After that, I built a bash audit script and ran it every 10 to 15 minutes for the rest of the competition. 11+ sweeps, all clean. Red team never got back in.

The thing that hurt my score the most was the SSH certificate check. I was the only one on the team whose SSH-CERT scoring was going on and off, so it was likely caused by the missing domain users. A teammate on the AD side confirmed that we were supposed to have 40 domain users but I only had 24 on my box.

**Key moments from the day**

- Ran inventory on kvigindis immediately after connecting: users, ports, services, SSH config.
- Found `AuthorizedPrincipalsFile none` as the root cause of the compromise.
- Identified and removed all four persistence mechanisms before red team could use them again.
- Built a custom audit script during the event and ran it repeatedly to confirm the box stayed clean.
- Hardened the Docker container: confirmed PHP dangerous functions (exec, system, passthru, shell_exec) were all disabled to prevent PHP2Shell attacks.
- Helped a teammate patch an SSHole vulnerability on their Alpine machine by telling them to update OpenSSH.
- Struggled with SSH-CERT scoring going on and off, likely due to only having 24 of the expected 40 domain users.
  - Wrote an IR report documenting the full incident with screenshots and evidence for future reference.

## 4. Lessons Learned

**What went well**

- I found and cleaned all four persistence mechanisms before red team could reestablish access.
- I identified the root cause (AuthorizedPrincipalsFile none) and patched it correctly with per user principal files.
- I built an audit script during the event and ran it consistently, confirming 11+ clean sweeps with no reinfection.
- I hardened the firewall to only the ports I needed and verified the Docker container's PHP config was locked down.
- Compared to Invitational 1, I was much more independent and methodical. I checked one thing at a time instead of guessing.

**What went wrong and why**

| Problem | Root cause |
|---------|------------|
| Did not catch the SSH config vulnerability immediately | I was not familiar with AuthorizedPrincipalsFile or how SSH certificate authentication works under the hood. The fix was changing `AuthorizedPrincipalsFile none` to `AuthorizedPrincipalsFile /etc/ssh/auth_principals/%u` and creating a file per user in that directory containing their allowed principal name. This meant only certificates issued with the correct principal could log in as that user |
| SSH-CERT scoring kept going on and off | I was the only one with this issue, so it was likely caused by only having 24 of the expected 40 domain users on my box |
| Low uptime score (20%) | Time lost to the initial compromise and the ongoing cert renewal issue |
| Slow initial response to the compromise | Should have run audit checks immediately at competition start before doing anything else |
| SSH session kept timing out | Did not set up tmux at the start, so disconnects interrupted my work |
| Had to ask for the root password multiple times | No password cheat sheet prepared ahead of time |
| Was stubborn about documentation | I resisted writing IR reports and injects during the event, but documentation is part of the competition scoring and helps the whole team coordinate |

## 5. What I'm Working On Next

**Before the next competition**

- Write and test a Linux audit script ahead of time so it is ready to drop on any box in the first 5 minutes. Include checks for users, sudoers, crontabs, systemd timers, bashrc backdoors, SSH keys, and recently modified services.
- Write a Windows audit script in PowerShell covering local admins, scheduled tasks, new services, recent logon events, firewall rules, and startup programs.
- Build a first 5 minutes script that changes the root/admin password, runs the full audit, enables the firewall, sets up tmux, and installs the Wazuh agent.
- Build a continuous monitor script that watches /etc/passwd, /etc/sudoers.d/, systemd, and auth.log in the background via tmux.
- Study SSH certificate authentication, AuthorizedPrincipalsFile, and Step CA so I understand the full cert renewal lifecycle before the event.
- Practice coordinating cross box issues with teammates so handoffs on shared infrastructure (like a CA server) are faster.

**Habits to change**

- Run audit checks first, before anything else, at competition start.
- Set up tmux immediately after connecting so SSH disconnects do not interrupt work.
- Keep a password cheat sheet ready so I do not waste time asking for credentials.
- Always `cat` suspicious scripts and configs before deleting them so I have evidence for the IR report.
- Verify expected user counts early. If the box should have 40 users and I only see 24, flag it immediately.
- Ask the router team to block known red team IPs at the perimeter as soon as they are identified.
- Put real effort into documentation, IR reports, and injects. They are worth points and help the team.
