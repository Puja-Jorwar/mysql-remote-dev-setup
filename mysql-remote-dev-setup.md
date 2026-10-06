# Shared MySQL Development Database Setup

## Goal

Allow a colleague's backend running on another Mac to temporarily connect to the MySQL database running on my Mac.

### Architecture

```text
Colleague's Mac
      |
      | Same office Wi-Fi / VPN
      |
      | MySQL connection
      | YOUR_IP:3306
      ↓
My Mac
      |
      ↓
MySQL 8.0
      |
      ↓
vit database
```

---

# 1. Check MySQL on my Mac

### Check MySQL version

```bash
mysql --version
```

### Check MySQL service

```bash
brew services list | grep mysql
```

Expected:

```text
mysql@8.0 started
```

### Find my Mac's IP

```bash
ipconfig getifaddr en0
```

Example:

```text
192.168.0.123
```

This IP can change when connecting to a different Wi-Fi network, so check it again when working from the office.

---

# 2. Configure MySQL for network connections

MySQL configuration file:

```text
/opt/homebrew/etc/my.cnf
```

Open it:

```bash
nano /opt/homebrew/etc/my.cnf
```

Configuration:

```ini
# Default Homebrew MySQL server config
[mysqld]
bind-address = 0.0.0.0
mysqlx-bind-address = 127.0.0.1
skip-log-bin
```

Save:

```text
Ctrl + O
Enter
Ctrl + X
```

Restart MySQL:

```bash
brew services restart mysql@8.0
```

---

# 3. Verify MySQL is listening on port 3306

Run:

```bash
lsof -nP -iTCP:3306 -sTCP:LISTEN
```

Expected:

```text
mysqld ... TCP *:3306 (LISTEN)
```

Also check:

```bash
mysql -h 127.0.0.1 -P 3306 -u root -p
```

Then:

```sql
SHOW VARIABLES LIKE 'bind_address';
```

Expected:

```text
0.0.0.0
```

---

# 4. Create a separate debugging user

DO NOT give the colleague the MySQL root password.

Connect as root:

```bash
mysql -h 127.0.0.1 -P 3306 -u root -p
```

Create a dedicated user:

```sql
CREATE USER 'pujas_gig'@'%' IDENTIFIED BY 'YOUR_STRONG_TEMPORARY_PASSWORD';
```

Grant access only to the required database:

```sql
GRANT ALL PRIVILEGES ON vit.* TO 'pujas_gig'@'%';
```

Then:

```sql
FLUSH PRIVILEGES;
```

Verify:

```sql
SHOW GRANTS FOR 'pujas_gig'@'%';
```

Expected:

```text
GRANT ALL PRIVILEGES ON `vit`.* TO `pujas_gig`@`%`
```

### IMPORTANT

Never commit the actual password to GitHub.

Use a new temporary password and share it separately with the colleague.

---

# 5. Check both Macs are on the same network

On my Mac:

```bash
ipconfig getifaddr en0
```

On colleague's Mac:

```bash
ipconfig getifaddr en0
```

Example:

```text
My Mac:         192.168.1.20
Colleague Mac:  192.168.1.35
```

They should normally be reachable within the same network.

At home, a mobile hotspot may isolate devices or place them on different networks, so test this at the office network/VPN.

---

# 6. Test the MySQL port from colleague's Mac

On colleague's Mac:

```bash
nc -vz -w 5 YOUR_IP 3306
```

Example:

```bash
nc -vz -w 5 192.168.1.20 3306
```

Success:

```text
Connection to 192.168.1.20 port 3306 [tcp/mysql] succeeded!
```

If it fails, DO NOT randomly change MySQL settings.

Check:

- Both Macs are on the same network
- Correct IP address
- Office Wi-Fi/VPN allows device-to-device communication
- Firewall/network restrictions

---

# 7. Connect from colleague's Mac

Once port 3306 is reachable:

```bash
mysql -h YOUR_IP -P 3306 -u pujas_gig -p
```

Example:

```bash
mysql -h 192.168.1.20 -P 3306 -u pujas_gig -p
```

Enter the temporary password.

If successful:

```text
Welcome to the MySQL monitor.
mysql>
```

Then:

```sql
USE vit;
```

Check tables:

```sql
SHOW TABLES;
```

---

# 8. Connect the backend

The colleague's backend should use YOUR Mac's IP instead of localhost.

### Before

```text
localhost:3306
```

### After

```text
YOUR_IP:3306
```

Example:

```text
192.168.1.20:3306
```

The database configuration conceptually becomes:

```text
Host:     192.168.1.20
Port:     3306
Database: vit
Username: pujas_gig
Password: temporary password
```

### Important

Do NOT use:

```text
localhost
```

in the colleague's backend.

For her backend, `localhost` means HER own Mac.

---

# 9. Security

This setup is intended for a trusted development network.

Do NOT:

- expose port 3306 to the public internet
- configure router port forwarding for MySQL
- share the root password
- commit database passwords to GitHub
- use this setup as a production database architecture

Use a dedicated database user.

After debugging is finished, remove the temporary user.

```sql
DROP USER 'pujas_gig'@'%';
```

Then verify:

```sql
SHOW GRANTS FOR 'pujas_gig'@'%';
```

The user should no longer exist.

---

# 10. Troubleshooting

## Error 2003

```text
Can't connect to MySQL server on 'YOUR_IP:3306'
```

Check:

```bash
lsof -nP -iTCP:3306 -sTCP:LISTEN
```

Then from the colleague's Mac:

```bash
nc -vz -w 5 YOUR_IP 3306
```

If the port times out, investigate the network/firewall/VPN rather than the MySQL username/password.

---

## Error 1045

```text
Access denied for user
```

This usually means the MySQL server was reached, but authentication failed.

Check the username/password and MySQL account:

```sql
SELECT user, host FROM mysql.user;
```

Check permissions:

```sql
SHOW GRANTS FOR 'pujas_gig'@'%';
```

---

## Ping fails

```bash
ping -c 4 YOUR_IP
```

A failed ping does not automatically mean MySQL is unavailable because ICMP can be blocked.

Test the actual MySQL port instead:

```bash
nc -vz -w 5 YOUR_IP 3306
```

---

# Current Status

## Completed

- [x] MySQL 8.0 installed on my Mac
- [x] MySQL service running
- [x] Mac IP identified
- [x] `bind-address` changed to `0.0.0.0`
- [x] MySQL listening on TCP port 3306
- [x] Database identified: `vit`
- [x] Dedicated user created: `pujas_gig`
- [x] User granted access to `vit`
- [x] Local MySQL configuration completed

## Pending

- [ ] Connect both Macs to the same office network/VPN
- [ ] Find the new IP of my Mac
- [ ] Find colleague's IP
- [ ] Test port 3306 from colleague's Mac
- [ ] Connect colleague's MySQL client
- [ ] Connect colleague's backend
- [ ] Test the application
- [ ] Remove temporary database user after debugging

---

# Quick Tomorrow Checklist

### My Mac

```bash
ipconfig getifaddr en0
```

```bash
lsof -nP -iTCP:3306 -sTCP:LISTEN
```

### Colleague's Mac

```bash
ipconfig getifaddr en0
```

```bash
nc -vz -w 5 YOUR_IP 3306
```

If successful:

```bash
mysql -h YOUR_IP -P 3306 -u pujas_gig -p
```

Then:

```sql
USE vit;
SHOW TABLES;
```

---

## Important Reminder

The IP address can change when changing Wi-Fi networks.

Therefore, always run:

```bash
ipconfig getifaddr en0
```

again when moving from home to the office.

Never store the actual MySQL password in this README or in GitHub.
