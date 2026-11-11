# MongoDB 8.0 on WSL2 (Ubuntu 24.04): Install and Access from Windows

This guide installs MongoDB Community Edition 8.0 in **Ubuntu 24.04 on WSL2** and connects to it from **Windows** using `127.0.0.1:27017`. It does **not** expose MongoDB to the internet or other LAN devices.

> **Storage:** If your Ubuntu WSL distribution is installed at `D:\WSL\Ubuntu-24.04`, its Linux filesystem (including MongoDB's default `/var/lib/mongodb`) resides inside its virtual disk on **D:**. Confirm the location before proceeding. WSL itself still uses some C: space.

## 1. Check WSL and launch Ubuntu 24.04 (Windows PowerShell)

```powershell
wsl --list --verbose
wsl -d Ubuntu-24.04
```

Inside Ubuntu, verify:

```bash
cat /etc/os-release | grep VERSION_ID
```

Expected: `VERSION_ID="24.04"`.

To check the distro's storage location from **Windows PowerShell**:

```powershell
Get-ChildItem HKCU:\Software\Microsoft\Windows\CurrentVersion\Lxss |
  ForEach-Object {
    $d = Get-ItemProperty $_.PSPath
    [PSCustomObject]@{ Name=$d.DistributionName; Location=$d.BasePath }
  } | Format-Table -AutoSize
```

## 2. Install MongoDB 8.0 (Ubuntu Bash)

```bash
sudo apt update
sudo apt install -y curl gnupg

curl -fsSL https://pgp.mongodb.com/server-8.0.asc \
  | sudo gpg --dearmor -o /usr/share/keyrings/mongodb-server-8.0.gpg

echo 'deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.0 multiverse' \
  | sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list

sudo apt update
sudo apt install -y mongodb-org
mongod --version
```

If the signing-key file already exists, `gpg` may prompt before overwriting it. Review the prompt instead of blindly repeating the command.

Official instructions: https://www.mongodb.com/docs/v8.0/tutorial/install-mongodb-on-ubuntu/

## 3. Enable systemd and start MongoDB

Check Ubuntu's init system:

```bash
ps -p 1 -o comm=
```

If it says `systemd`, run:

```bash
sudo systemctl enable --now mongod
sudo systemctl status mongod --no-pager
```

If it does **not** say `systemd`, edit `/etc/wsl.conf`:

```bash
sudo nano /etc/wsl.conf
```

Add `systemd=true` under an existing `[boot]` section (or create the section):

```ini
[boot]
systemd=true
```

In **Windows PowerShell** (not Ubuntu Bash):

```powershell
wsl --shutdown
wsl -d Ubuntu-24.04
```

Then, in Ubuntu Bash:

```bash
sudo systemctl enable --now mongod
sudo systemctl status mongod --no-pager
```

## 4. Bind MongoDB to localhost only

Edit MongoDB's configuration in Ubuntu:

```bash
sudo nano /etc/mongod.conf
```

Ensure the existing `net` section contains (do not create duplicate `net` sections):

```yaml
net:
  port: 27017
  bindIp: 127.0.0.1
```

Restart and check:

```bash
sudo systemctl restart mongod
sudo ss -lntp | grep 27017
mongosh --host 127.0.0.1 --port 27017 --eval 'db.adminCommand({ ping: 1 })'
```

A successful ping includes `{ ok: 1 }`.

> Localhost binding is for **same-computer development**. Do not change `bindIp` to `0.0.0.0` to expose an unauthenticated MongoDB server.

## 5. Connect from Windows

WSL2 generally forwards Linux services bound to localhost to Windows localhost. In **Windows PowerShell**:

```powershell
Test-NetConnection 127.0.0.1 -Port 27017
Get-NetTCPConnection -LocalPort 27017 -State Listen -ErrorAction SilentlyContinue |
  Select-Object LocalAddress, LocalPort, OwningProcess
```

Expected: `TcpTestSucceeded : True`.

**Port conflict:** If MongoDB is also installed as a native Windows service, port 27017 might be served by Windows rather than WSL. Inspect first:

```powershell
Get-Service *mongo*
```

Only if you deliberately want to stop the native Windows instance, find its exact service name and stop that service; do not delete its database files.

In **MongoDB Compass** on Windows, use:

```text
mongodb://127.0.0.1:27017
```

For a local Node.js or Python project:

```dotenv
MONGODB_URI=mongodb://127.0.0.1:27017/myapp
```

## 6. Test by creating a database (Ubuntu Bash)

```bash
mongosh
```

Then inside `mongosh`:

```javascript
use myapp
db.users.insertOne({ name: 'Test User', createdAt: new Date() })
db.users.find()
```

Refresh Compass to see the `myapp` database.

## 7. Troubleshooting

**`ECONNREFUSED 127.0.0.1:27017` inside Ubuntu:**

```bash
sudo systemctl status mongod --no-pager
sudo journalctl -u mongod -n 50 --no-pager
sudo ss -lntp | grep 27017
```

If the service has just started, wait a few seconds and retry the ping. If it exits, inspect the journal for the underlying error.

**`Get-NetTCPConnection: command not found`:** This is a **PowerShell** command. Run it in Windows PowerShell, not Ubuntu Bash.

**Windows cannot connect but Ubuntu can:** Check whether the native Windows MongoDB service owns port 27017; restart WSL with `wsl --shutdown` and reopen Ubuntu if necessary. Check your WSL networking settings if localhost forwarding is disabled.

**MongoDB stops after WSL shuts down:** `systemctl enable mongod` starts it when the Ubuntu distribution starts; it does not keep WSL running indefinitely.

## 8. Security notes

- This configuration is for **localhost access only** and does not expose MongoDB to other computers.
- MongoDB's default local development setup may have authentication disabled. Do not open port 27017 on your router or firewall for public access.
- For LAN or remote access, first configure MongoDB users, authentication, appropriate firewall rules, and ideally a secure tunnel or VPN.
- Back up important databases before removing or moving your WSL distribution.
