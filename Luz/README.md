# Luz — HackMyVM Writeup

## 1. Reconnaissance

We start with an `nmap` scan using `-p-` to scan all ports, and `-Pn` to skip host discovery (no ping):

```bash
nmap -p- -Pn 192.168.1.59
```

![nmap scan showing ports 22 and 80 open](images/01-nmap-scan.png)

We'll focus on port 80 (HTTP).

## 2. Web Enumeration

We use `gobuster` to search for interesting files and directories (php, md, txt, db, etc.):

```bash
gobuster dir -u http://192.168.1.59 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html
```

![gobuster scan on root directory revealing /admin, /assets, /database, etc.](images/02-gobuster-root.png)

We find the `/admin` directory, so we run `gobuster` again against that folder:

```bash
gobuster dir -u http://192.168.1.59/admin -w /usr/share/wordlists/dirb/common.txt -x php,txt,html
```

![gobuster scan on /admin revealing users.php, orders.php, menu.php, etc.](images/03-gobuster-admin.png)

The full results of the scan:

![full gobuster results on /admin directory](images/06-gobuster-admin-full.png)

## 3. Credential Discovery

We navigate to `http://192.168.1.59/admin/users.php`. We run `curl -i` against the page looking for any hints or comments in the HTML source, but found nothing useful there.

![users.php page showing Administrator (hadmin) and Staff accounts](images/04-users-php.png)

So we try the classic "username = password" approach for both accounts, which turns out to work.

## 4. Authenticated Access & Privilege Escalation (Web App)

Logging in as `staff` gives access to the admin panel (Home, Orders, Menu, Category List):

![staff dashboard with menu/orders overview](images/05-admin-home.png)

When clicking on "Orders", the URL changes to:

```
http://192.168.1.59/admin/index.php?page=orders
```

We try changing the `page` parameter to `users`:

```
http://192.168.1.59/admin/index.php?page=users
```

This gives us access to user management, including a **"New user"** button — even though we're only logged in as `staff`:

![index.php?page=users showing New user button available to staff](images/07-page-users-param.png)

We create a new user with **Admin** privileges (name/username: `Opavoloko`):

![new user form with Admin role selected](images/08-new-user-form.png)

## 5. Gaining a Reverse Shell

Now authenticated as an admin, we find a page with a file upload field. We test it and confirm that the application accepts `.php` files — this is a classic unrestricted file upload vulnerability.

We upload a PHP reverse shell and confirm the upload succeeds:

![POST request to ajax.php?action=save_settings returning 200 OK](images/09-upload-request.png)

We start a listener on our machine:

```bash
nc -lvnp 4444
```

![netcat listener waiting on port 4444](images/10-nc-listener.png)

We then trigger the uploaded shell by requesting it in the browser, and get a connection back:

![reverse shell caught as www-data](images/11-reverse-shell-caught.png)

## 6. Shell Stabilization

Once we catch the shell, we stabilize the TTY:

```bash
script /dev/null -c bash
^Z
stty raw -echo; fg
reset xterm
export TERM=xterm SHELL=bash
```

![stabilized shell with full TTY as www-data](images/12-shell-stabilized.png)

## 7. User Flag

We move up the directory tree to find `user.txt`:

```bash
cd ..
cd ..
cd ..
ls
# fos  user.txt
cat user.txt
```

![locating and reading user.txt](images/13-user-flag-location.png)

The flag is submitted on the HackMyVM platform:

![user flag captured on HackMyVM](images/14-submit-flag-user-captured.png)

## 8. Privilege Escalation — Part 1 (SUID binary → user `aelis`)

We look for SUID binaries to escalate our privileges:

```bash
find / -perm -4000 -ls 2>/dev/null
```

![full list of SUID binaries including /usr/bin/bsd-csh owned by aelis](images/15-suid-search-full.png)

One binary, `/usr/bin/bsd-csh`, is owned by `aelis` (not root) and has the SUID bit set. We check [GTFOBins](https://gtfobins.github.io/) for `csh` and find a privilege escalation technique using the `-b` flag:

```bash
/usr/bin/bsd-csh -b
```

![using bsd-csh -b to spawn a shell as aelis](images/16-bsd-csh-exploit.png)

We're now the user `aelis`.

## 9. Lateral Movement via SSH Keys

We enter `/home/aelis` and look around:

![aelis home directory listing](images/17-aelis-home-dir.png)

We find the `.ssh` directory with an `authorized_keys` file:

![.ssh directory containing authorized_keys](images/18-aelis-ssh-dir.png)

We generate an SSH key pair on our attacking machine (if we don't already have one):

```bash
ssh-keygen -t rsa -b 4096
```

We then paste our **public** key into `authorized_keys` using `nano` (via the `csh` SUID shell):

![pasting our public key into authorized_keys](images/19-authorized-keys-paste.png)

With the key in place, we log in directly via SSH as `aelis`:

```bash
ssh 192.168.1.59 -l aelis
```

![SSH login as aelis using the newly added key](images/20-ssh-login-aelis.png)

## 10. Privilege Escalation — Part 2 (Enlightenment CVE → root)

Now with a bit more privilege, we repeat the SUID search and notice a set of files related to **Enlightenment** (`enlightenment_ckpasswd`, `enlightenment_system`, `enlightenment_sys`). This points to a known local privilege escalation vulnerability affecting Enlightenment: **CVE-2022-37706**.

![SUID search as aelis showing the enlightenment binaries](images/21-suid-search-aelis.png)

We download/create the exploit script from this repository:
[CVE-2022-37706-LPE-exploit](https://github.com/MaherAzzouzi/CVE-2022-37706-LPE-exploit/blob/main/exploit.sh)

```bash
cd /home/aelis/
nano exploit.sh
# (paste the exploit code)
```

![creating exploit.sh with nano](images/22-exploit-sh-nano.png)

We give it execute permissions and run it:

```bash
chmod +x exploit.sh
./exploit.sh
```

![running the Enlightenment exploit and getting a root shell](images/23-exploit-execution-root.png)

## 11. Root Flag

```bash
cd /
ls
cd root
ls
# root.txt  snap
cat root.txt
```

![navigating to /root and reading root.txt](images/24-root-flag-location.png)

![HackMyVM congratulations screen — VM pwned](images/25-vm-pwned.png)

---

## Summary

| Stage | Technique |
|---|---|
| Recon | `nmap -p- -Pn` |
| Enumeration | `gobuster` directory brute-forcing |
| Initial access | Weak credentials (username = password) |
| Privilege escalation (web) | Parameter manipulation (`index.php?page=users`) allowing a staff account to create admin users |
| Foothold | Unrestricted file upload → PHP reverse shell |
| User → `aelis` | SUID `bsd-csh` binary abused via GTFOBins (`-b` flag) |
| Persistence | SSH key added to `authorized_keys` |
| `aelis` → root | CVE-2022-37706 (Enlightenment local privilege escalation) |

**Tools used:** `nmap`, `gobuster`, `curl`, browser dev tools, `nc`/`ncat`, GTFOBins, `ssh-keygen`, public CVE exploit script.

*Hack and Fun.*
