# win-ssh-rvsproxy
Tunneling Socks proxy through Windows.

This document presents a Red Team approach for SSH tunneling to a Socks proxy on a compromised Windows computer from an external attacker-controlled Linux host.  How the Windows computer is compromised is not covered here; the assumption is that you have a way to run commands on the victim host.

The following diagram shows the scenario.  An external attacker system (A) is running sshd to allow tunnel creation, and will also run python SimpleHTTPServer to serve an ephemeral SSH key to allow the Windows computer (B) to log in.  The Windows target (B) will execute the built-in ssh.exe command line utility and create a Socks proxy, through which the attacker (A) can access other systems protected behind the firewall, such as the web server (C).

![alt text](https://github.com/billchaison/win-ssh-rvsproxy/blob/main/01.png)

