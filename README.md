# Docker Compose - MongoDB & Postgres on WSL2

This guide shows how to install Docker Engine and Docker Compose directly in **Ubuntu 24.04 on WSL 2**, then build and start the Compose project at:

- Windows: `F:\Project\Docker\local\docker-compose.yml`
- WSL: `/mnt/f/Project/Docker/local/docker-compose.yml`

> Run PowerShell commands in **Windows PowerShell** and Bash commands in **Ubuntu WSL**. You do **not** need Docker Desktop.

## 1. Verify WSL 2 (PowerShell)

```powershell
wsl -l -v
```

Your distribution should be `Ubuntu-24.04` with `VERSION` set to `2`.

Launch it:

```powershell
wsl -d Ubuntu-24.04
```

## 2. Verify systemd (Ubuntu)

```bash
ps -p 1 -o comm=
```

If it prints `systemd`, proceed. Otherwise, edit `/etc/wsl.conf`:

```bash
sudo nano /etc/wsl.conf
```

Add `systemd=true` under an existing `[boot]` section, or create that section:

```ini
[boot]
systemd=true
```

Save and close the editor. In **PowerShell**, restart WSL:

```powershell
wsl --shutdown
wsl -d Ubuntu-24.04
```

Check again that `ps -p 1 -o comm=` prints `systemd`.

## 3. Install prerequisites (Ubuntu)

```bash
sudo apt update
sudo apt install -y ca-certificates curl
```

## 4. Add Docker's official Ubuntu repository (Ubuntu)

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

```bash
sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF_REPO
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF_REPO
```

```bash
sudo apt update
apt-cache policy docker-ce
```

A nonempty `Candidate` version indicates the package is available.

## 5. Install and start Docker (Ubuntu)

```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo systemctl is-active docker
sudo docker --version
sudo docker compose version
sudo docker run --rm hello-world
```

The service should be `active`, and the test should print `Hello from Docker!`.

## 6. Optional: run Docker without sudo (Ubuntu)

```bash
sudo usermod -aG docker "$USER"
newgrp docker
```

Then verify:

```bash
docker info
docker compose version
```

**Security:** The `docker` group grants root-equivalent privileges. If you prefer not to grant these permissions, skip this step and prefix Docker commands with `sudo`.

## 7. Build and start your Compose project (Ubuntu)

Navigate to the directory containing the Compose file:

```bash
cd /mnt/f/Project/Docker/local
ls -la
```

Validate the Compose configuration:

```bash
docker compose config
```

> The validation output may display environment variables or secrets. Review it before sharing publicly.

Build and start services in the background:

```bash
docker compose up -d --build
```

If you skipped Step 6, use `sudo docker compose ...` instead.

Check containers and logs:

```bash
docker compose ps
docker compose logs -f
```

Press **Ctrl+C** to stop following logs; this does not stop containers started with `-d`.

### Run from any directory

```bash
docker compose -f /mnt/f/Project/Docker/local/docker-compose.yml up -d --build
```

### Useful commands

| Task | Command |
| --- | --- |
| Validate configuration | `docker compose config` |
| Build only | `docker compose build` |
| Build and start in background | `docker compose up -d --build` |
| Show running services | `docker compose ps` |
| Follow logs | `docker compose logs -f` |
| Restart services | `docker compose restart` |
| Stop services | `docker compose stop` |
| Stop and remove containers/networks | `docker compose down` |
| List images | `docker images` |

**Caution:** `docker compose down -v` also removes named volumes, which can delete persisted application data. Do not use it unless that is intended.

## Troubleshooting

**`docker: command not found`**: Complete Step 5 and reopen Ubuntu if necessary.

**`Cannot connect to the Docker daemon`**:

```bash
sudo systemctl status docker
sudo systemctl start docker
```

**`permission denied ... docker.sock`**: Use `sudo docker ...` or complete Step 6, then open a fresh shell.

**`no configuration file provided`**: Run `pwd` and `ls -la`; make sure you're in `/mnt/f/Project/Docker/local` or specify `-f`.

**Build fails due to missing paths or files**: Review the `build:` context, Dockerfile paths, `.env` files, and bind mounts in `docker-compose.yml`.

**Performance on `/mnt/f`**: Windows-mounted files can be slower for Linux container builds and bind mounts. For better performance, consider storing source code in Ubuntu's Linux filesystem (such as `~/projects`) and adjust any paths accordingly.

**Note:** `docker compose up -d --build` builds only services that specify a build configuration. Services using only `image:` generally pull or use the named image.

## Official reference

Docker Engine on Ubuntu: https://docs.docker.com/engine/install/ubuntu/
