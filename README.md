# TryHackMe – Jump

A practical, step-by-step walkthrough of the **Jump** room on TryHackMe.

This walkthrough covers the complete attack path, from initial access through Linux privilege escalation to retrieving the final root flag.

## Topics Covered

- Nmap reconnaissance and FTP enumeration
- Anonymous FTP access
- Initial access through a reverse shell
- Linux user and group enumeration
- Scheduled task enumeration using pspy64
- SSH public-key authentication
- PATH hijacking
- Sudo misconfigurations
- Linux privilege escalation

## Attack Path

```text
Anonymous FTP
     |
     v
recon_user
     |
     v
dev_user
     |
     v
monitor_user
     |
     v
ops_user
     |
     v
Root flag
```

## Walkthrough

See [Jump.md](Jump.md) for the complete walkthrough, commands, explanations, and flags.

**Platform:** TryHackMe  
**Room:** Jump  
**Difficulty:** Linux Privilege Escalation

*Created for learning and documenting practical penetration-testing techniques.*
