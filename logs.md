# Logs
In this file, I showcase what the logs over on Wazuh showed and didn't show during each major phase of the attack.
# Setup
The setup for the logs was pretty straightforward. The relevant log files tracked were: `auth.log` (for SSH), `access.log` (for Apache), `dpkg.log` (for DEB packages), and `vsftpd.log` (for FTP). No log files were aggregated from other sources, and Wazuh served as a single pane of glass. This means anything not properly displayed on Wazuh would be missed from the SOC side, showing the potential danger in relying on one centralized dashboard that assumes full visibility.

# NMAP
When performing a basic scan of our DVWA host, no logs were generated. This is because these logs will necessitate a network-based logging solution like Suricata, which many corporations lack.
Although someone scanning your server isn't necessarily harmful or needs a defensive reaction, it can indicate a potential attacker. That said, harmless scans are nearly impossible to differentiate from harmful ones because of the amount of network traffic popular websites receive.

# DVWA: Dictionary Attack and File Upload
Upon requesting the DVWA login page and beginning to try a few common credential combinations, we can see our Wazuh events page displaying the following:

<img width="783" height="400" alt="image" src="https://github.com/user-attachments/assets/08e7d854-3e20-4714-8288-41a76c0fa999" />

Examining one of the events more closely, we see:

<img width="1363" height="694" alt="image" src="https://github.com/user-attachments/assets/68bb5bcd-1b6b-4494-9190-8f794ec10a71" />

This `POST` request tells us that `192.168.100.1` is probably trying out a pair of credentials. We can also see a `302` redirection to `/login.php`, which likely means that the attempt failed, and via our previous picture, it's obvious that the individual is doing this quite a few times. However, the logged event does not show us what _combinations_ are being tried, which is pivotal in a real environment where someone might be trying methods like SQL or Command Injection.

Soon after, when looking into the event I highlighted, we see:

<img width="1345" height="114" alt="image" src="https://github.com/user-attachments/assets/f3053490-f987-4d4a-9661-0cd0f3951deb" />

Notice how the `GET` request is being made to `/index.php` and not `/login.php`. This is likely indicates a successful login, letting us know the person is now logged into our DVWA web-app.

When navigating to the 'File Upload' page, we have more `GET` requests displaying this. And when attempting to upload, we see:

<img width="1366" height="572" alt="image" src="https://github.com/user-attachments/assets/76b7393a-a5c9-4185-9ce5-a66ab6c02c3a" />

There is no error code in the first request even though we know it failed, and this is because we still received a valid `200 OK` HTTP response. This can come off like an issue with the SIEM, but the latter can only track what the log files report, so if the web server itself does not issue an error code after a failed attempt, it won't be properly reported.
Right after, there's another `POST` request, showing that the individual is trying to upload again. So they might be trying an exploit!

Shortly after, we find a `GET` request to `/hackable/uploads/`, quickly followed by an `nmap` installation:

<img width="1367" height="553" alt="image" src="https://github.com/user-attachments/assets/f2c5d561-c1f5-4998-bb56-caa5fecaac38" />

We can also see many associated `dpkg` events:

<img width="1580" height="591" alt="image" src="https://github.com/user-attachments/assets/c2d84b95-666b-487a-b235-a284dc61572b" />

Only now do we find a `GET` request explicitly showing `php-reverse-shell.php` which is our uploaded file. This filename should raise alerts immediately.

<img width="1376" height="446" alt="image" src="https://github.com/user-attachments/assets/635f5b3b-98b4-45e3-bb8a-e92616f24a32" />

This demonstrates that log ingestion timings can throw off an analyst's ability to sequentially order events. Nevertheless, it's quite clear now that somebody executed a shell, installed `nmap`, and is probably planning to begin lateral movement. That said, there is no log for the command `ip a` since it's not associated with any log file we registered.

Soon after, we see a log for our `ftp` command:

<img width="1376" height="538" alt="image" src="https://github.com/user-attachments/assets/ff002733-4880-48c6-b7d7-2fa5e0387510" />

# Internal 1: FTP, SSH and Privilege Escalation
We haven't yet touched base on Internal 1, but we have established an FTP connection to it. From its side, we see:

<img width="1372" height="548" alt="image" src="https://github.com/user-attachments/assets/0d5b15b8-5f9a-490d-8ae8-322b1669330b" />

Notice that the timing lines up with the one from our log over at DVWA, and this helps analyze what's going on. That said, timeouts, errors, retries etc. can muddy the logs and make it more difficult to piece them together. We can also see that there are no FTP logs for downloading the `info.txt` file; we can only observe the attempt and subsequent authentication.

Right after our successful connection, we see an SSH event correlated to logging into `john`'s account:

<img width="1372" height="417" alt="image" src="https://github.com/user-attachments/assets/2b9131fe-844b-4302-86ca-591cf7130304" />

We can see the connection is being made from our DVWA machine. This should serve as a red flag, since `john`--an _internal_ employee--won't be connecting from a web server.

After a bunch of logs related to file creation (mostly due to SSH authentication), we find:

<img width="1372" height="344" alt="image" src="https://github.com/user-attachments/assets/3274febc-5bba-4545-aca2-effe0ba850d3" />

Seeing this preceded by the `john` SSH connection showcases possible privilege escalation, more specifically of the vertical kind (since we're elevating privileges).

# Final Notes
The collected logs demonstrate the visibility, or lack thereof, when it comes to using a SIEM. Although not exhaustive, many logs here, like the one with `php-reverse-shell`, the installation of unwanted packages, the FTP connection from a web-server, etc. would've caused SOC staff to intervene, likely stopping the attack. Nevertheless, they could also be drowned out in noise, and evasion is possible since these can happen in a few minutes at most, which is why several factors must be prioritized, like full visibility, quick detection and response, and automation.

