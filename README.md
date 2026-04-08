# jenkins_cicd

Local Jenkins in Docker for CI/CD against your GitHub repositories (for example [basketball_statistics](https://github.com/jreyesromero/basketball_statistics)).

## Prerequisites

- Docker available on your Mac: [Docker Desktop](https://www.docker.com/products/docker-desktop/) **or** [Colima](https://github.com/abiosoft/colima) (and the Docker CLI), with the engine running before you build.

## Commands (from this repository)

**Important:** Any `docker compose …` command must be run from the directory that contains `docker-compose.yml` (this repository). If your terminal is open in another project (for example `basketball_statistics`), Compose will report `no configuration file provided: not found`. Either `cd` here first, or use plain `docker …` commands that target the container by name (see the initial password step below).

Open a terminal and run:

```bash
cd /Users/julianre/workspace/personal/jenkins_cicd
```

Build the image (installs Jenkins LTS and plugins from `plugins.txt`):

```bash
docker compose build
```

### If `docker compose build` fails: TLS / certificate error (Colima or corporate network)

If you see an error like:

```text
tls: failed to verify certificate: x509: certificate signed by unknown authority
```

when pulling `jenkins/jenkins` from Docker Hub, the Docker daemon (often **inside the Colima Linux VM**) does not trust the HTTPS certificate chain. That usually happens on a **corporate VPN or network** that performs **SSL inspection** (traffic is re-signed with a custom root CA that your Mac trusts but the VM does not).

1. **Quick check:** Run `docker compose build` again while **off the VPN** or on another network (for example a phone hotspot). If it succeeds, you need the corporate CA inside Colima, not a change to this repo.
2. **Confirm from the VM:**  
   `colima ssh -- curl -vI https://registry-1.docker.io/v2/`  
   If you see certificate errors here, fix trust in the VM (next step).
3. **Install your organization’s root CA in the Colima VM:** Obtain the root certificate as a `.pem` file (from IT or your security team, or export the issuing root from Keychain). Then, from your Mac:
   - Copy the file into the VM (for example `scp` to a path Colima can read, or create the file after `colima ssh`).
   - Inside the VM, as a user with `sudo`:
     - `sudo cp your-company-root.pem /usr/local/share/ca-certificates/company-root.crt`
     - `sudo update-ca-certificates`
   - Exit the VM and restart Colima: `colima stop` then `colima start`.
4. Retry:

```bash
docker compose build
docker compose up -d
```

Also verify your **system date and time** are correct; large clock skew can break TLS.

Do **not** mark Docker Hub as an “insecure registry” to work around this; add the proper CA instead.

Start Jenkins in the background:

```bash
docker compose up -d
```

Show logs (optional, until you see “Jenkins is fully up and running”):

```bash
docker compose logs -f jenkins
```

Press `Ctrl+C` to stop following logs; Jenkins keeps running.

Read the one-time **administrator password** (pick one):

- **From this repository** (so Compose finds `docker-compose.yml`):

```bash
cd /Users/julianre/workspace/personal/jenkins_cicd
docker compose exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

- **From any directory** (uses the container name `jenkins` from `docker-compose.yml`; run `docker ps` if you need to confirm the name):

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Open the UI in a browser:

```text
http://localhost:8080
```

Paste the initial password, continue the setup wizard (install suggested plugins or skip and manage plugins later), and create your admin user when prompted.

## After Jenkins is running

1. **GitHub access:** Manage Jenkins → Credentials → (global) → Add Credentials. Use a [GitHub personal access token](https://github.com/settings/tokens) (classic: `repo` scope is enough for private repos; public repos can use anonymous read in some setups).
2. **Multibranch job:** New Item → **Multibranch Pipeline** → Branch Sources → **GitHub** → Credentials → pick your credential → Repository HTTPS URL or “Repository by name” (owner/repo).
3. **Pipeline in the app repo:** Add a `Jenkinsfile` to the application repository (see `examples/basketball_statistics/Jenkinsfile`). Commit and push; the multibranch scan will pick up branches.

If GitHub cannot reach your laptop (no public URL), configure the multibranch job to **Scan Multibranch Pipeline Triggers** → **Periodically if not otherwise run** (for example every 5 minutes), or trigger **Scan Multibranch Pipeline Now** manually from the job page.

## Stop and remove

Stop Jenkins (run from this repository, same as other `docker compose` commands):

```bash
cd /Users/julianre/workspace/personal/jenkins_cicd
docker compose down
```

Stop and **delete** Jenkins data (jobs, plugins, credentials stored in the volume):

```bash
cd /Users/julianre/workspace/personal/jenkins_cicd
docker compose down -v
```

If you are not in this directory, you can stop the container by name: `docker stop jenkins` (data in the volume is kept unless you remove the volume with `docker volume rm …`).

## Optional: Docker builds inside Jenkins

The default `docker-compose.yml` does not mount the host Docker socket. If you later need `docker build` from pipelines, you can extend Compose with a socket mount and install the Docker CLI in a custom image; that is more convenient on Linux than on Docker Desktop for Mac, so this setup keeps the first-run path simple.

## Files

| File | Purpose |
|------|--------|
| `Dockerfile` | Jenkins LTS + plugins from `plugins.txt` + Python 3 for sample pipelines on the built-in node |
| `docker-compose.yml` | Service, ports `8080` / `50000`, named volume `jenkins_home` |
| `examples/basketball_statistics/Jenkinsfile` | Sample pipeline for the basketball project |
