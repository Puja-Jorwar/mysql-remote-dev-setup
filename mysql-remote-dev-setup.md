# Shared MySQL Development Database Setup

## Goal

Allow a colleague's backend running on another Mac to temporarily connect to the MySQL database running on my Mac for development and debugging.

This avoids repeatedly creating and sharing large database dumps when a colleague needs to reproduce or debug an issue using the same development data.

---

# Architecture

```text
Colleague's Mac
      |
      | Same office Wi-Fi / trusted network
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

> **Tested successfully:** This setup was verified between two Macs connected to the same office Wi-Fi. The colleague's Mac successfully reached MySQL port `3306` and authenticated using the dedicated database user.

---

# 1. Check MySQL on My Mac

## Check MySQL version

```bash
mysql --version
```

Example:

```text
mysql  Ver 8.0.x for macos...
```

## Check MySQL service

```bash
brew services list | grep mysql
```

Expected:

```text
mysql@8.0 started
```

## Find My Mac's IP address

```bash
ipconfig getifaddr en0
```

Example:

```text
192.168.x.x
```

The IP address can change when connecting to a different Wi-Fi network.

Therefore, always check the IP again when moving between networks such as home and office.

---

# 2. Configure MySQL for Network Connections

The Homebrew MySQL configuration file is:

```text
/opt/homebrew/etc/my.cnf
```

Open it:

```bash
nano /opt/homebrew/etc/my.cnf
```

Configure the `[mysqld]` section as follows:

```ini
# Default Homebrew MySQL server config
[mysqld]
bind-address = 0.0.0.0
mysqlx-bind-address = 127.0.0.1
skip-log-bin
```

Save the file:

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

# 3. Verify MySQL Is Listening on Port 3306

Run:

```bash
lsof -nP -iTCP:3306 -sTCP:LISTEN
```

Expected:

```text
mysqld ... TCP *:3306 (LISTEN)
```

This confirms that MySQL is accepting TCP connections on port `3306`.

You can also test the local MySQL connection:

```bash
mysql -h 127.0.0.1 -P 3306 -u root -p
```

After connecting, check:

```sql
SHOW VARIABLES LIKE 'bind_address';
```

Expected:

```text
0.0.0.0
```

---

# 4. Identify the Required Database

Connect to MySQL:

```bash
mysql -h 127.0.0.1 -P 3306 -u root -p
```

Check databases:

```sql
SHOW DATABASES;
```

For this setup, the required database is:

```text
vit
```

---

# 5. Create a Separate Debugging User

## Do not use the MySQL root account

The colleague should NOT receive the MySQL root password.

Create a separate user specifically for development/debugging.

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

Verify the permissions:

```sql
SHOW GRANTS FOR 'pujas_gig'@'%';
```

Expected:

```text
GRANT ALL PRIVILEGES ON `vit`.* TO `pujas_gig`@`%`
```

### Important

Never commit the actual database password to GitHub.

Use a temporary password and share it with the colleague through an appropriate private channel.

---

# 6. Check Both Macs Are on the Same Network

## On My Mac

```bash
ipconfig getifaddr en0
```

## On Colleague's Mac

```bash
ipconfig getifaddr en0
```

Example:

```text
My Mac:          192.168.x.x
Colleague Mac:   192.168.x.x
```

Both machines need to be able to communicate with each other.

Office Wi-Fi or an approved company VPN should be used.

### Note about mobile hotspots

A mobile hotspot may isolate devices or place them on different network configurations.

For example, two devices may receive different IP ranges and therefore may not be able to communicate directly.

If this happens, test using the office network or an approved VPN instead.

---

# 7. Test Network Connectivity

Before testing MySQL authentication, test whether the colleague's Mac can reach My Mac.

## Optional ping test

On the colleague's Mac:

```bash
ping -c 4 YOUR_IP
```

A successful result indicates network connectivity.

However, ping is not a reliable MySQL test because ICMP may be blocked even when TCP connections are allowed.

## Recommended MySQL port test

Run from the colleague's Mac:

```bash
nc -vz -w 5 YOUR_IP 3306
```

Example:

```bash
nc -vz -w 5 192.168.x.x 3306
```

Successful result:

```text
Connection to 192.168.x.x port 3306 [tcp/mysql] succeeded!
```

If this succeeds, the colleague's Mac can reach the MySQL server.

---

# 8. Connect to MySQL From the Colleague's Mac

Once port `3306` is reachable:

```bash
mysql -h YOUR_IP -P 3306 -u pujas_gig -p
```

Example:

```bash
mysql -h 192.168.x.x -P 3306 -u pujas_gig -p
```

Enter the temporary password when prompted.

Successful connection:

```text
Welcome to the MySQL monitor.
mysql>
```

---

# 9. Verify Access to the `vit` Database

After connecting:

```sql
USE vit;
```

Then:

```sql
SHOW TABLES;
```

The tables from the `vit` database should be displayed.

You can also verify the current database:

```sql
SELECT DATABASE();
```

Expected:

```text
vit
```

---

# 10. Configure the Colleague's Backend

The colleague's backend must connect to **My Mac's IP address**, not `localhost`.

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
192.168.x.x:3306
```

The database configuration should conceptually be:

```text
Host:      YOUR_IP
Port:      3306
Database:  vit
Username:  pujas_gig
Password:  temporary password
```

## Important

Do NOT use:

```text
localhost
```

in the colleague's backend.

For the colleague's backend:

```text
localhost = colleague's own Mac
```

The database is running on **my Mac**, so the backend must use my Mac's network IP address.

---

# 11. Grails Backend Configuration

For a Grails project using `application.yml`, the development datasource can be configured to point to the MySQL server running on the other Mac.

Example:

```yaml
dataSource:
    pooled: true
    jmxExport: true
    driverClassName: com.mysql.cj.jdbc.Driver
    username: pujas_gig
    password: YOUR_TEMPORARY_PASSWORD
    dialect: org.hibernate.dialect.MySQL8Dialect

environments:
    development:
        dataSource:
            dbCreate: none
            url: jdbc:mysql://YOUR_IP:3306/vit?useSSL=false
```

Replace:

```text
YOUR_IP
```

with the IP address of the Mac running MySQL.

For example:

```yaml
url: jdbc:mysql://192.168.x.x:3306/vit?useSSL=false
```

### Why `dbCreate: none`?

When connecting to an existing shared development database, we generally do not want Grails to automatically modify or recreate the database schema.

The exact `dbCreate` setting should still follow the project's existing development configuration and team practice.

---

# 12. Test the Backend

After updating the development datasource:

1. Stop the backend if it is already running.
2. Update the datasource configuration.
3. Start the backend again.
4. Watch the application logs.
5. Verify that the application successfully connects to MySQL.
6. Test the API/page/functionality related to the issue being debugged.

If the backend starts successfully and can read data from `vit`, the remote database configuration is working.

---

# 13. Troubleshooting

## Error 2003

```text
Can't connect to MySQL server on 'YOUR_IP:3306'
```

First check the MySQL server:

```bash
lsof -nP -iTCP:3306 -sTCP:LISTEN
```

Expected:

```text
mysqld ... TCP *:3306 (LISTEN)
```

Then test from the colleague's Mac:

```bash
nc -vz -w 5 YOUR_IP 3306
```

If the port test fails, investigate:

- Both Macs are on the same network
- Correct IP address is being used
- MySQL is running
- MySQL is listening on port `3306`
- Office Wi-Fi allows device-to-device communication
- Approved VPN/network configuration
- Firewall or network restrictions

Do not randomly change MySQL settings before identifying whether the problem is MySQL or the network.

---

# 14. Error 1045 — Access Denied

Example:

```text
Access denied for user 'pujas_gig'@'...' 
```

This usually means:

- The MySQL server was successfully reached
- But authentication failed

Check the username and password.

From the MySQL server:

```sql
SELECT user, host
FROM mysql.user;
```

Check permissions:

```sql
SHOW GRANTS FOR 'pujas_gig'@'%';
```

Expected access:

```text
GRANT ALL PRIVILEGES ON `vit`.* TO `pujas_gig`@`%`
```

---

# 15. Ping Fails

If:

```bash
ping -c 4 YOUR_IP
```

fails, this does not automatically mean MySQL is unavailable.

ICMP traffic can be blocked by the network.

Test the actual MySQL port instead:

```bash
nc -vz -w 5 YOUR_IP 3306
```

If port `3306` succeeds, MySQL connectivity is available even if ping fails.

---

# 16. Error: Connection Refused

If:

```bash
nc -vz -w 5 YOUR_IP 3306
```

returns connection refused, check the MySQL server:

```bash
lsof -nP -iTCP:3306 -sTCP:LISTEN
```

Also check that MySQL is running:

```bash
brew services list | grep mysql
```

Restart if necessary:

```bash
brew services restart mysql@8.0
```

---

# 17. Security

This setup is intended for temporary development/debugging on a trusted company network.

## Do

- Use a dedicated MySQL user
- Grant access only to the required database
- Use a temporary password
- Share credentials privately
- Get appropriate team/manager approval before exposing a development database over the office network
- Remove access when debugging is finished
- Restore the MySQL configuration when remote access is no longer required

## Do NOT

- Expose MySQL port `3306` to the public internet
- Configure router port forwarding for MySQL
- Share the MySQL root password
- Commit database passwords to GitHub
- Store credentials directly in source control
- Use this setup as a production database architecture
- Leave temporary remote access enabled unnecessarily

---

# 18. Cleanup After Debugging

Once the debugging session is finished, remove the temporary database user.

Connect to MySQL:

```bash
mysql -h 127.0.0.1 -P 3306 -u root -p
```

Drop the user:

```sql
DROP USER 'pujas_gig'@'%';
```

Verify:

```sql
SELECT user, host
FROM mysql.user
WHERE user = 'pujas_gig';
```

The query should return no rows.

---

# 19. Restore Local-Only MySQL Access

After remote debugging is finished, change the MySQL configuration back to local-only access.

Open:

```bash
nano /opt/homebrew/etc/my.cnf
```

Change:

```ini
bind-address = 0.0.0.0
```

back to:

```ini
bind-address = 127.0.0.1
```

Keep:

```ini
mysqlx-bind-address = 127.0.0.1
skip-log-bin
```

The configuration should become:

```ini
# Default Homebrew MySQL server config
[mysqld]
bind-address = 127.0.0.1
mysqlx-bind-address = 127.0.0.1
skip-log-bin
```

Restart MySQL:

```bash
brew services restart mysql@8.0
```

Verify:

```bash
lsof -nP -iTCP:3306 -sTCP:LISTEN
```

MySQL should no longer be listening on all network interfaces.

---

# 20. Current Status

## Completed

- [x] MySQL 8.0 installed on My Mac
- [x] MySQL service running
- [x] Mac IP identified
- [x] `bind-address` changed to `0.0.0.0`
- [x] MySQL listening on TCP port `3306`
- [x] Database identified: `vit`
- [x] Dedicated debugging user created
- [x] User granted access to `vit`
- [x] Both Macs connected to the same office Wi-Fi
- [x] Colleague's Mac can reach port `3306`
- [x] Remote MySQL login tested successfully
- [x] Remote MySQL authentication tested successfully
- [x] Ready to connect the colleague's backend

## Remaining

- [ ] Get appropriate approval for the temporary shared database setup
- [ ] Connect colleague's Grails backend
- [ ] Test the application
- [ ] Remove temporary database user after debugging
- [ ] Restore `bind-address` to `127.0.0.1`

---

# 21. Quick Office Checklist

## My Mac

Get the current IP:

```bash
ipconfig getifaddr en0
```

Check MySQL:

```bash
brew services list | grep mysql
```

Check port:

```bash
lsof -nP -iTCP:3306 -sTCP:LISTEN
```

---

## Colleague's Mac

Test connectivity:

```bash
nc -vz -w 5 YOUR_IP 3306
```

Connect to MySQL:

```bash
mysql -h YOUR_IP -P 3306 -u pujas_gig -p
```

Verify database:

```sql
USE vit;
```

Check tables:

```sql
SHOW TABLES;
```

---

# 22. Important Reminders

### IP address can change

The Mac's IP address may change when switching Wi-Fi networks.

Always run:

```bash
ipconfig getifaddr en0
```

again when moving between networks.

### Never commit credentials

Never put the actual database password in:

- `README.md`
- `application.yml`
- GitHub
- Git commits
- Screenshots
- Public documentation

Use environment variables or another approved secret-management approach for real projects.

### Temporary setup

This approach is useful for occasional development/debugging, but it should not become the team's permanent database architecture.

For regular team development, a shared development/staging database or another approved centralized environment is generally more appropriate.

### Network security

This setup should only be used on an approved/trusted company network or VPN.

Never expose MySQL directly to the public internet.

---

# 23. Example Workflow

The complete workflow is:

```text
1. Get approval
        ↓
2. Connect both Macs to approved office network/VPN
        ↓
3. Find My Mac's IP
        ↓
4. Configure MySQL bind-address
        ↓
5. Restart MySQL
        ↓
6. Create temporary database user
        ↓
7. Grant access to vit database
        ↓
8. Test port 3306 from colleague's Mac
        ↓
9. Test remote MySQL login
        ↓
10. Configure colleague's Grails backend
        ↓
11. Start backend
        ↓
12. Reproduce/debug the issue
        ↓
13. Remove temporary database user
        ↓
14. Restore bind-address to 127.0.0.1
        ↓
15. Restart MySQL
```

---

## Final Checklist

Before finishing the debugging session:

```text
[ ] Backend debugging completed
[ ] Temporary MySQL user removed
[ ] Database password no longer shared unnecessarily
[ ] bind-address restored to 127.0.0.1
[ ] MySQL restarted
[ ] Remote access disabled
[ ] No credentials committed to GitHub
```
