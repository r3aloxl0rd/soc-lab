# Attack Scenario

I will go through each attack phase and showcase the different techniques and attack vectors. Screenshots will be included. For the sake of this project, I will be using a basic attack framework of:
- Reconnaisance
- Exploitation
- Lateral Movement
- Actions on Objectives (Conceptual)

# Reconaissance
Initially, as the host, we only have visibility into the DVWA machine given that it's a web-app. We know its IP is `192.168.100.2` (if this was a real target, depending on the type of penetration test it is, you might have access to a public IP instead of a private one).

First, we send out a basic (no flags) NMAP scan using `nmap 192.168.100.2`. We use a basic scan to avoid causing too much noise (you can also use the `-Pn` flag to not send discovery pings since they might trigger alerts, but it will cause `nmap` to assume the host(s) is active).
These are the results:

<img width="790" height="255" alt="Screenshot_20260421_134950" src="https://github.com/user-attachments/assets/9c198468-7ab0-4619-883b-9075c9176067" />

We can see there's an open HTTP port leading to the DVWA login page.

<img width="490" height="492" alt="Screenshot_20260424_233227" src="https://github.com/user-attachments/assets/ec9b9f02-505d-4fcb-9718-6bdbcca343e6" />

There's also an open SSH port, and an attacker can begin brute-forcing credentials, but that is usually inefficient and risky when you don't even possess a username to try.
However, trying credentials on a web-app login page is more realistic, especially since you might get in through other methods like SQL Injection.
Trying out a few different basic combinations, like `admin:admin` and `root:toor`, we land on the correct one: `admin:password`!

Now, we can see multiple different potentially-exploitable avenues. The most interesting one is File Upload.

<img width="290" height="753" alt="Screenshot_20260423_112743" src="https://github.com/user-attachments/assets/6efd1915-1933-4636-bc0b-993db8907571" />

This is because file uploads usually lead to reverse shells, if the system is vulnerable enough. This can quickly allow us to take control of the server, and this is why it's targeted.

We can tell the website is using PHP (either via analyzing the URL path or by checking the output of an extension like [Wappalyzer](https://www.wappalyzer.com/)). So, we look for a good PHP reverse-shell, and attempt to upload it. I used [this one](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php).
When we try uploading it, though, we get an error, because only image file types are allowed, like PNG or JPEG.

<img width="1073" height="252" alt="image" src="https://github.com/user-attachments/assets/0538de6c-1911-44be-a385-1884e942687a" />

In order to circumvent this, assuming lackluster security measures, we can modify our packet to make it seem like we're sending an image file of the desired type. I used [Burp Suite](https://portswigger.net/burp), and here's the modified request (I highlighted what was altered):

<img width="776" height="514" alt="image" src="https://github.com/user-attachments/assets/cf84b88b-5cd0-4247-9c3f-b28d6f1fcfc9" />

You can see the `Content-Type` header says `image/png` instead of what it should say which is `application/x-php`.

# Exploitation
Now, our request goes through, and the file is uploaded!

<img width="1112" height="328" alt="image" src="https://github.com/user-attachments/assets/e31aef79-cb09-4321-804b-93a42b34ae53" />

We can see the directory in which it is uploaded seems to be at `/hackable/uploads/`, and we can confirm the file is there by navigating to `http://192.168.100.2/hackable/uploads/`, so that's where we'll launch it from.

But before doing that, we must run `nc -lvp <PORT>`. This is because the shell I used creates a connection back to an active listener on a custom port, so we must launch this listener.
Once it is is active, we click on our file from the aforementioned directory, and we can see that our listener has initiated a shell.

<img width="1237" height="254" alt="image" src="https://github.com/user-attachments/assets/1a4b59b8-8a08-4f72-be04-db82144ba479" />

Running `whoami`, we see that we're the `www-data` user, which is usually the default user for managing Apache web-server configurations.
Reading `/etc/passwd` lets us know there's another interesting user called `administrator`, and we can attempt guessing his password. But for this lab, I will prioritize pivoting towards the internal network.

Running `ip a`, we find:

<img width="1229" height="533" alt="image" src="https://github.com/user-attachments/assets/04878e40-b811-499e-a430-8e11082feda0" />

We can see that the machine has an IP for a different subnet than the one we initially scanned.

We attempt to use `nmap` to enumerate what devices exist on this subnet and are visible, but the `nmap` package isn't installed. To do so, we must use `sudo` to run an `apt` command.
However, we first check if `www-data` is capable of doing so, so we run `sudo -l` (this gives us an overview of their `sudo` privileges, if they have any).

We see: `User www-data may run the following commands on DVWA:
    (ALL) NOPASSWD: ALL`

This means that they're able to run `sudo` commands without us needing to input their password. So, we go ahead and run `sudo apt install nmap -y`.

Once `nmap` is installed, we scan the subnet using `nmap 192.168.200.0/24`. We spot an internal machine (Internal 1) under `192.168.200.2`, and it has both an FTP and SSH port running.

<img width="627" height="171" alt="image" src="https://github.com/user-attachments/assets/a5daae6a-034b-48bc-bdc4-a8dc5efb7845" />

Again, we can try some combinations for SSH, but instead, we'll first attempt an anonymous login to FTP using `sudo ftp anonymous@192.168.200.2` (in this lab, `ftp` was already installed. If it wasn't, we'd install it like we did `nmap`). The `sudo` part is necessary for being able to download whatever FTP-accessible files we find.

Our shell is minimal (for now), so graphical feedback is lackluster at best, but we can see it worked because it didn't error out. When listing files (using `ls`), there's one called `info.txt`, and it contains a password combination of `john:password123`!

<img width="772" height="249" alt="image" src="https://github.com/user-attachments/assets/95c20c7b-2a06-42c8-86b6-04be2726f0dc" />


# Lateral Movement
Now, we can try out that SSH port. But, before doing that, we should upgrade our shell to something terminal-based; otherwise, SSH-ing won't work. There are different ways to do this, but a reliable one is `python3 -c 'import pty; pty.spawn("/bin/bash")'`. That command uses the `python3` binary to import a pseudo-terminal module and uses it launch an interactive Bash shell.

<img width="942" height="557" alt="image" src="https://github.com/user-attachments/assets/a282e0f5-4e16-44a7-996f-367e1061d62f" />

And we are officially inside the internal network! We can try escalating privileges by seeing if `root` also has the same password as `john`, and for this lab, it does.

<img width="228" height="115" alt="image" src="https://github.com/user-attachments/assets/d82caf94-83c7-4104-873a-6ceedbe92446" />

Further pivoting should be straightforward from here on out, depending on what other internal machines are visible from our current pivot point.

# Actions on Objectives
When on DVWA, we could've sabotaged the web application or attempted reading non-public files, and when on Internal 1, we could potentially exfiltrate sensitive data.

# Final Notes
The process of attacking this environment is fairly rudimentary and depended on some misconfigurations that are realistically rare nowadays, such as anon FTP access, extremely weak passwords for employee accounts, and an interface connecting a DMZ machine to an internal one. Nonetheless, it demonstrates that if there are multiple vulnerabilities, they can easily be chained together by an outside party to establish a foothold on your network.

Check the `logs.md` file for corresponding log information about what the SIEM solution displayed throughout this attack scenario.
