# Base IMIS Local Deployment Guide

This guide describes how to deploy Base IMIS on the same server where Ansible is running. It covers requirements, local inventory, playbook order, verification, SSH access, Nginx Proxy Manager access, and GeoServer access through an SSH tunnel.

## 1. Basic Requirements

The web application and database are preferably deployed on same server.

### 1.1 Server Requirements

| Requirement      | Recommendation                                                                             |
| ---------------- | ------------------------------------------------------------------------------------------ |
| Operating system | Ubuntu 22.04 LTS or newer. Automation also supports Debian 12 or newer.                    |
| Disk space       | At least 100 GB initially, plus space for GIS data, media, backups, images, and logs.      |
| RAM              | At least 8 GB. More may be required for large GIS datasets or higher traffic.              |
| CPU              | At least 4 CPU cores. More may be required for larger workloads.                           |
| Network          | At least 100 Mbps and approximately 500 GB to 1 TB monthly transfer, depending on traffic. |

### 1.2 Software Requirements

| Software         | Requirement                                                                               |
| ---------------- | ----------------------------------------------------------------------------------------- |
| Web server       | Nginx is used by the deployment. Apache is possible with modifications.                   |
| PHP              | PHP 8 or later. The current Laravel playbook defaults to PHP 8.2.35.                      |
| Database         | PostgreSQL 14                                                                             |
| Spatial database | PostgreSQL 14 with PostGIS 3.5; `postgis` and `postgis_topology` extensions are enabled.  |
| Routing          | pgRouting 4.0, enabled by the PostgreSQL playbook.                                        |
| GeoServer        | GeoServer 2.28.2.                                                                         |
| Containers       | Docker Engine and Docker Compose plugin, installed from Docker's official APT repository. |

### 1.3 Windows Access Tools

When the deployment engineer connects from a Windows computer, install the following tools before starting:

| Tool     | Purpose                                                                              |
| -------- | ------------------------------------------------------------------------------------ |
| PuTTYgen | Create or convert the SSH private and public key pair used for server access.        |
| PuTTY    | Open an SSH terminal to the server and run the setup commands and Ansible playbooks. |
| WinSCP   | Connect to the server over SFTP to upload, download, and manage deployment files.    |

Use the SSH public key created with PuTTYgen on the server. Keep the private key secure and never commit it to Git or upload it to the deployment repository.

#### 1.3.1 Download Putty and puttygen

Open the supplied PuTTY and PuTTYgen download page:

https://puttygen.com/download-putty#PuTTY_for_windows

The download page displays the **Alternative Binary Files for PuTTY Windows** list, choose the 64-bit x86 files for most modern Windows computers:

| File           | Download   | Purpose                                                            |
| -------------- | ---------- | ------------------------------------------------------------------ |
| `putty.exe`    | 64-bit x86 | PuTTY SSH terminal used to connect to the server and run commands. |
| `puttygen.exe` | 64-bit x86 | PuTTYgen key-generation tool used to create the Ed25519 SSH key.   |

Scroll down the download page until the `puttygen.exe` section is visible. Download the 64-bit x86 `puttygen.exe` file as well as the 64-bit x86 `putty.exe` file.

For the complete process of installing and running PuTTY on Windows, use the same reference page:

https://puttygen.com/download-putty#PuTTY_for_windows

#### 1.3.2 Create an Ed25519 SSH Key with PuTTYgen

The supplied page is a third-party download/reference page. Before running the installer or standalone executable, verify that the download is the expected PuTTY release and scan it with the organization's security tools. The official PuTTY release page remains the preferred source when available.

1. Download `puttygen.exe` if you are using the standalone PuTTYgen download, or download the PuTTY Windows installer if PuTTYgen is included in the package.
2. Open **PuTTYgen** from the Windows Start menu or run `puttygen.exe`.
3. Under **Parameters**, select **Ed25519**.
4. Click **Generate** and move the mouse over the blank area until key generation completes.
5. Enter a strong **Key passphrase(optional)** and confirm it.
6. Click **Save private key** and save the file in a secure location, for example:

   ```text
   C:\Users\YOUR_USERNAME\.ssh\base-imis-ed25519.ppk
   ```

7. In the **Public key for pasting into OpenSSH authorized_keys file** box, select and copy the complete public key text. Copy it as one line beginning with `ssh-ed25519`.
8. Keep the `.ppk` private key on the desktop computer only. Never paste the private key into the server, Git, WinSCP, or the deployment repository.

#### 1.3.3 Connect with PuTTY

Use the same supplied download page as the reference for the PuTTY Windows package:

https://puttygen.com/download-putty#PuTTY_for_windows

1. Download the PuTTY Windows installer. Use the 64-bit installer for most modern Windows computers; use the 32-bit installer if required by the computer.
2. Run the installer with Windows administrator permissions.
3. Accept the welcome screen and keep the default installation directory unless there is a specific reason to change it.
4. Keep the standard PuTTY features selected. Add a desktop shortcut if it will be used frequently.
5. Complete the installation and click **Finish**.

#### 1.3.4 Install and Configure WinSCP

1. Download WinSCP from the official website:

   https://winscp.net/eng/downloads.php

2. Run the installer.
3. Accept the license agreement.
4. Select the recommended installation type and complete the installation.
5. Open WinSCP and select **New Site**.
6. Configure the site as follows:

   | WinSCP setting | Value                                                                   |
   | -------------- | ----------------------------------------------------------------------- |
   | File protocol  | SFTP                                                                    |
   | Host name      | Server IP address                                                       |
   | Port number    | Current SSH port, initially the provider port or `2224` after hardening |
   | User name      | `root` initially, or `imis_user` after SSH hardening                    |
   | Password       | The password for the selected user, if password login is enabled        |

#### 1.3.5 Open PuTTY from WinSCP

After the WinSCP session is connected:

1. Open the **Commands** menu.
2. Select **Open in PuTTY**, or press `Ctrl+P`.
3. WinSCP opens PuTTY using the saved server connection settings.
4. Use the PuTTY terminal to run commands and playbooks. Use WinSCP for file transfer and file management.

WinSCP uses SFTP over SSH. Therefore, the SSH username, port, password, key, and firewall access must be correct before WinSCP can connect.

### 1.4 Deployment Engineer Requirements

- Linux server administration with Ubuntu or Debian.
- Docker and Docker Compose.
- Networking and port configuration.
- Firewall rules.
- PostgreSQL and PostGIS.
- GeoServer.
- Shell scripting.
- PHP and Laravel.
- NPM, Git, JavaScript, jQuery, and OpenLayers.

## 2. Required Services and Ports

| Service                   |  Port | Requirement or usage                                                       |
| ------------------------- | ----: | -------------------------------------------------------------------------- |
| SSH                       |  2224 | Configured administrative SSH port in the current setup.                   |
| HTTP                      |    80 | Public web traffic.                                                        |
| HTTPS                     |   443 | Secure public web traffic.                                                 |
| Nginx Proxy Manager admin |    81 | Normally bound to localhost and accessed through an SSH tunnel.            |
| PostgreSQL host           | 15432 | Default host port, normally bound to localhost for local-only access.      |
| GeoServer host            | 18082 | Current host port, mapped to container port 8080.                          |
| Docker networks           |   N/A | PostgreSQL, GeoServer, Nginx, and Laravel must use matching network names. |

## 3. Run the Initial Control Node Setup

The initial setup script is stored in the `ansible-playbook-server-setup` folder of the GitHub repository.
GitHub repository:

https://github.com/base-imis/ansible_playbook_server_setup

### Connect to the server with WinSCP

1. Open WinSCP and select **New Site**.
2. Select **SFTP** as the file protocol.
3. Enter the server address, SSH port, username, and authentication method supplied for the deployment.
4. Click **Login** and verify that the remote file panel displays the server directory.

Initial control-node script:

https://github.com/base-imis/ansible_playbook_server_setup/01-setup-control-node-safe.sh

The script installs the control-node tools and then clones the Ansible playbook repository. A separate bootstrap directory is not required. Create the script directly in the current home directory:

```bash
sudo nano 01-setup-control-node-safe.sh
```

Open the GitHub file at `ansible-playbook-server-setup/01-setup-control-node-safe.sh`, copy only its raw script content, and paste it into the local file `/root/01-setup-control-node-safe.sh` opened in nano. Do not copy the Markdown code fences or the filename itself. Save with `Ctrl+O`, press `Enter`, and exit with `Ctrl+X`.

Make the file executable and check its syntax:

```bash
sudo chmod +x 01-setup-control-node-safe.sh
```

```bash
sudo bash -n 01-setup-control-node-safe.sh
```

No output from `bash -n` means the syntax check passed. Run the script:

```bash
sudo ./01-setup-control-node-safe.sh
```

#### Control-node setup prompts, values, and purpose

| Prompt                          | Example/default value                                            | Purpose                                                            |
| ------------------------------- | ---------------------------------------------------------------- | ------------------------------------------------------------------ |
| Upgrade installed packages now? | `Y` or `y` to skip                                               | Controls whether the script performs a full package upgrade first. |
| Deployment repository URL       | `https://github.com/base-imis/ansible_playbook_server_setup.git` | Selects the Ansible repository to clone.                           |
| Repository directory            | `/root/server-setup-ansible`                                     | Selects where the cloned Ansible repository is stored.             |

When the script asks for the deployment repository URL, use:

```text
https://github.com/base-imis/ansible_playbook_server_setup.git
```

When it asks for the repository directory, use the default or choose:

```text
/root/server-setup-ansible
```

### What `01-setup-control-node-safe.sh` does

1. Creates the repository directory.
2. Installs Ansible, Python, Git, the OpenSSH client, and sudo.
3. Clones the repository.
4. After the script finishes, enter the cloned Ansible directory:

```bash
cd /root/server-setup-ansible
```

## 4. Run the Initial SSH Script

After `01-setup-control-node-safe.sh` completes, the Ansible repository has already been cloned and contains the existing `02-ssh.sh` file. Do not create or copy `02-ssh.sh` again. Enter the cloned repository and run the existing file:

```bash
cd /root/server-setup-ansible
```

```bash
sudo chmod +x 02-ssh.sh
```

```bash
sudo ./02-ssh.sh
```

#### Initial SSH script prompts, values, and purpose

| Prompt                                                    | Example/default value                     | Purpose                                                  |
| --------------------------------------------------------- | ----------------------------------------- | -------------------------------------------------------- |
| Add a public SSH key to `/root/.ssh/authorized_keys` now? | `y` or `N`                                | Controls whether a PuTTYgen/OpenSSH public key is added. |
| Public key or file path                                   | Complete `ssh-ed25519` key or `.pub` path | Supplies the public key used for SSH authentication.     |
| Add another public SSH key?                               | `y` or `N`                                | Controls whether another public key is added.            |

This script prepares the root SSH directory and public-key files on the control server. It does not connect to or modify a separate managed server.

When the script displays:

```text
Do you want to add a public SSH key to /root/.ssh/authorized_keys now? [y/N]:
```

Enter `y` if you want to use the Ed25519 public key created with PuTTYgen. The script then asks for the **public key or file path**:

- Paste the complete public key copied from PuTTYgen, beginning with `ssh-ed25519`; or
- Enter the path to a `.pub` file already on the server.

The script validates that the value is an OpenSSH public key, avoids adding an exact duplicate, and appends it to:

```text
/root/.ssh/authorized_keys
```

This public-key entry allows SSH key authentication for the root account while root login is permitted by the server's SSH configuration. The private key remains on the Windows computer and is selected in PuTTY or WinSCP.

Do not paste a private key into the prompt. A PuTTY private key normally has a `.ppk` extension and must never be placed in `authorized_keys`.

For a separate managed server, `02-ssh.sh` does not install the public key there. Use the provider console, an existing password-based SSH session, PuTTY, or WinSCP to add the public key to the managed user's `~/.ssh/authorized_keys` file. For the local deployment described in this guide, the control server and target are the same machine.

### What `02-ssh.sh` does

The script performs the following actions on the control server:

1. Creates or preserves `/root/.ssh` with secure permissions.
2. Creates or preserves `/root/.ssh/authorized_keys`.
3. Creates a dedicated Ed25519 Ansible key pair if one does not already exist:

   ```text
   /root/.ssh/id_ed25519_ansible
   /root/.ssh/id_ed25519_ansible.pub
   ```

4. Validates the private key, public key, and `authorized_keys` file.
5. Sets secure permissions on the SSH files.
6. Asks whether you want to add another public SSH key to `/root/.ssh/authorized_keys`.

## 5. Configure the Local Inventory

```bash
cd ~/server-setup-ansible
```

```bash
nano inventory.ini
```

Use this inventory for a local deployment:

```ini
[control]
localhost ansible_connection=local

[managed]
server1 ansible_host=SERVER_IP ansible_user=INITIAL_SSH_USER ansible_port=22


[hardening_target:children]
control
```

Remove or comment out remote placeholders such as:

For control server:

```ini
localhost ansible_connection=local
```

For managed server:

```ini
server1 ansible_host=SERVER_IP ansible_user=INITIAL_SSH_USER ansible_port=22
```

Test the inventory:

```bash
ansible-inventory --graph
```

```bash
ansible control -m ansible.builtin.ping
```

```bash
ansible managed -m ansible.builtin.ping
```

Expected result:

```text
localhost | SUCCESS
```

## 6. Deployment Order

```text
01 SSH hardening
02 Docker
03 Nginx Proxy Manager
04 Fail2ban
05 PostgreSQL/PostGIS/pgRouting
06 GeoServer
07 Shared Nginx initial setup
08 Laravel application
Verification
```

### 6.1 SSH Hardening

```bash
cd ~/server-setup-ansible
```

Run command:

```bash
ansible-playbook playbooks/01_linux_ssh_setup_harden.yml
```

### What `01_linux_ssh_setup_harden.yml` does

Configures the administrative user, hardens SSH, changes the SSH port if selected, restricts root login, configures firewall rules, and enables system hardening. Keep a recovery console available.

#### SSH hardening prompts, values, and purpose

| Prompt                               | Example/default value                 | Purpose                                                   |
| ------------------------------------ | ------------------------------------- | --------------------------------------------------------- |
| Initial SSH private-key path         | Remote execution only                 | Verifies access to the new administrative user.           |
| New non-root administrative username | Required; not `root`                  | Creates or selects the administrative account.            |
| New administrative-user password     | Required unless supplied by variables | Enables sudo access for the new administrator.            |
| New root password                    | Blank keeps the current password      | Changes the root password when requested.                 |
| Current SSH port                     | Current server SSH port               | Identifies the port used before hardening.                |
| New SSH port                         | Blank keeps the current port          | Selects the hardened SSH listening port.                  |
| Office public IPv4/CIDR              | Blank if not required                 | Allows trusted office SSH access through the firewall.    |
| Ansible control-server IPv4/CIDR     | Blank if not required                 | Allows the Ansible control server through the firewall.   |
| Allow inbound HTTP port 80?          | `yes`                                 | Allows public HTTP traffic.                               |
| Allow inbound HTTPS port 443?        | `yes`                                 | Allows public HTTPS traffic.                              |
| Lock the root account?               | `yes`                                 | Prevents direct root password login after verification.   |
| Delete root authorized keys?         | `yes`                                 | Removes root SSH keys after recovery access is confirmed. |
| Lock other interactive users?        | `no`                                  | Locks unused interactive Linux accounts.                  |
| Send logs to a remote syslog server? | `no`                                  | Enables remote system logging when required.              |
| Remote syslog server `HOST:PORT`     | Example: `siem-server.com:514`        | Identifies the remote logging destination.                |
| Pause before SSH/root-lock changes?  | `yes`                                 | Provides a safety checkpoint before high-risk changes.    |

#### Retest the WinSCP connection

After the SSH hardening playbook completes:

1. Keep the existing WinSCP session open until the new connection is verified.
2. If the SSH port changed, update the saved WinSCP site with the new port.
3. Confirm that the WinSCP username matches the new non-root administrative user.
4. Select the correct private key if key-based authentication is required.
5. Open a new WinSCP session and click **Login**.
6. Verify that the connection succeeds and the remote file panel opens normally.
7. Confirm that you can browse the required deployment directory and transfer files.
8. Close the old WinSCP session only after the new session has been tested successfully.

### 6.2 Docker Setup

Before running the Docker setup:

1. Log out of the root SSH/WinSCP session.
2. Log in again using the newly created non-root administrative user.
3. Confirm that the new user can run `sudo` commands.
4. Change to the cloned Ansible repository directory, for example:

```bash
  cd /home/imis_user/server-setup-ansible
```

Run command:

```bash
ansible-playbook playbooks/02_docker_setup_hardening.yml
```

### What `02_docker_setup_hardening.yml` does

Installs and hardens Docker and verifies the Docker daemon.

#### Docker setup prompts, values, and purpose

| Prompt                                     | Example/default value | Purpose                                         |
| ------------------------------------------ | --------------------- | ----------------------------------------------- |
| Install Docker audit rules?                | `yes`                 | Records Docker activity with audit rules.       |
| Restart Docker when configuration changes? | `yes`                 | Allows Ansible to restart Docker after changes. |
| Pause after each major section?            | `yes`                 | Stops for manual testing between Docker stages. |
| Maximum Docker JSON log-file size          | `50m`                 | Limits the size of each container log file.     |
| Maximum Docker JSON log files              | `5`                   | Sets the number of rotated log files retained.  |

#### Check for the installed Version

```bash
docker --version
```

```bash
docker compose version
```

### 6.3 Nginx Proxy Manager

Run command:

```bash
ansible-playbook playbooks/03_Nginx_proxy.yml
```

### What `03_Nginx_proxy.yml` does

Creates the Nginx Proxy Manager configuration and starts the container.

#### Nginx Proxy Manager prompts, values, and purpose

| Prompt                                      | Example/default value                 | Purpose                                                    |
| ------------------------------------------- | ------------------------------------- | ---------------------------------------------------------- |
| Nginx Proxy Manager install directory       | `/home/imis_user/nginx-proxy-manager` | Selects where NPM files are stored.                        |
| SSH port for the local admin tunnel         | `2224`                                | Sets the SSH port used in tunnel instructions.             |
| SSH username in tunnel instructions         | `imis_user`                           | Sets the username shown in access commands.                |
| Stop/remove existing NPM and rebuild?       | `no`                                  | Prevents accidental removal of an existing NPM deployment. |
| Manage Docker firewall rules for NPM ports? | `yes`                                 | Controls DOCKER-USER rules for ports 80, 81, and 443.      |
| Install NPM audit rules?                    | `yes`                                 | Adds lightweight audit rules for NPM activity.             |

#### Step 1: Remove the Old SSH Host Key

Before creating the tunnel, open Windows Command Prompt and remove the old SSH host-key entry:

```cmd
ssh-keygen -R "[217.216.59.198]:2224"
```

This removes the outdated entry from:

```text
C:\Users\User\.ssh\known_hosts
```

Run this step when the server was reinstalled, its SSH host key changed, or PuTTY/OpenSSH reports a host-key mismatch. You will be asked to accept the new host key when connecting again.

Local admin tunnel:

```cmd
ssh -p 2224 -N -L 8181:127.0.0.1:81 <server_name>@<server_ip>
```

Open 'http://127.0.0.1:8181'. Keep the tunnel terminal open.

Set up the Nginx Proxy Manager administrator account:

1. Enter the administrator name.
2. Enter the administrator email address.
3. Enter a strong administrator password.
4. Save the account details and sign in.

### 6.4 Fail2ban

Run command:

```bash
ansible-playbook playbooks/04_install_fail2ban.yml
```

### What `04_install_fail2ban.yml` does

Installs Fail2ban, protects SSH, configures the SSH jail, and whitelists trusted administrator IP addresses.

#### Fail2ban prompts, values, and purpose

| Prompt                               | Example/default value          | Purpose                                                 |
| ------------------------------------ | ------------------------------ | ------------------------------------------------------- |
| SSH port to protect                  | Blank auto-detects from `sshd` | Selects the SSH service monitored by Fail2ban.          |
| Admin/public IP or CIDR to whitelist | Required                       | Prevents the main administrator from being banned.      |
| Additional trusted IPs/CIDRs         | Optional, space-separated      | Adds other addresses to the allowlist.                  |
| Ban duration                         | `1h`                           | Sets how long failed-login addresses remain banned.     |
| Failure counting window              | `10m`                          | Sets the period used to count failures.                 |
| Maximum failed attempts              | `5`                            | Sets the threshold before banning an address.           |
| Pause for final SSH verification?    | `yes`                          | Confirms SSH access remains usable after configuration. |

### 6.5 PostgreSQL PostGIS and pgRouting

Run command:

```bash
ansible-playbook playbooks/05_install_postgres_hardened.yml
```

### What `05_install_postgres_hardened.yml` does

The PostgreSQL playbook performs the following steps:

1. Starts PostgreSQL.
2. Enables PostGIS, postgis_topology, and pgRouting.
3. Creates the required databases and users.
4. Configures `pg_hba.conf`.
5. Applies database access restrictions.
6. Verifies that PostgreSQL is ready.
7. Restores the application database backup into the created database after PostgreSQL is ready.
8. Deploys the Laravel application after the database restoration is complete.

#### PostgreSQL prompts, values, and purpose

| Prompt                                     | Example/default value        | Purpose                                                         |
| ------------------------------------------ | ---------------------------- | --------------------------------------------------------------- |
| PostgreSQL major version                   | `14`                         | Selects the PostgreSQL version to deploy.                       |
| Install directory                          | `/home/imis_user/postgres14` | Selects the host directory for PostgreSQL data and files.       |
| Default PostgreSQL database                | `defaultdb`                  | Selects the initial administration database.                    |
| DB admin user                              | `imis_user`                  | Selects the administrative PostgreSQL role.                     |
| DB admin password                          | Required and confirmed       | Authenticates the administrative role.                          |
| PostgreSQL superuser password              | Required and confirmed       | Sets the PostgreSQL superuser credential.                       |
| Host PostgreSQL port                       | `15432`                      | Selects the host-published database port.                       |
| Host bind IP                               | `127.0.0.1`                  | Restricts database publishing to local access when appropriate. |
| Access mode                                | `A`                          | Chooses public/admin or public plus private allowlists.         |
| Public/admin allowed IPs/CIDRs             | Optional                     | Restricts public or administrator database access.              |
| Private/internal allowed IPs/CIDRs         | Optional                     | Restricts private or internal database access.                  |
| Apply strict CONNECT isolation?            | `true`                       | Limits roles to explicitly permitted databases.                 |
| Disable postgres login after verification? | `true`                       | Disables the default `postgres` login after testing.            |
| Pause before reusing existing data?        | `yes`                        | Adds a safety checkpoint before using existing PostgreSQL data. |

Local admin tunnel:

```cmd
ssh -p 2224 -N -L 15432:127.0.0.1:15432 <server_name>@<server_ip>
```

Later database administration:

### Restore the database using pgAdmin

For a plain SQL file such as [`base_imis.sql`](https://github.com/base-imis/deployment_documentation/blob/master/database/base_imis.sql), use pgAdmin's **Query Tool**. Follow these steps:

1. Open the **[Base IMIS database documentation](https://github.com/base-imis/deployment_documentation/tree/master/database)** and select the SQL file required for the application.
2. Download [`base_imis.sql`](https://raw.githubusercontent.com/base-imis/deployment_documentation/master/database/base_imis.sql), or copy it to the Windows computer using WinSCP or another secure file-transfer tool.
3. Keep the PostgreSQL SSH tunnel open:

   ```cmd
   ssh -p 2224 -N -L 15432:127.0.0.1:15432 <server_name>@<server_ip>
   ```

4. Open pgAdmin on the Windows computer.
5. Right-click **Servers** and select **Register > Server**.
6. On the **General** tab, enter a name such as `Base IMIS PostgreSQL`.
7. On the **Connection** tab, enter:

   | Setting              | Value                                                |
   | -------------------- | ---------------------------------------------------- |
   | Host name/address    | `127.0.0.1`                                          |
   | Port                 | `15432`                                              |
   | Maintenance database | `defaultdb`                                          |
   | Username             | `imis_user`                                          |
   | Password             | The PostgreSQL password configured during deployment |

8. Click **Save** and connect to the server.
9. Expand **Databases** and select the target database created by the PostgreSQL playbook.
10. Right-click the target database and select **Query Tool**.
11. Open the downloaded `base_imis.sql` file in Query Tool, or paste its contents into the editor.
12. Click **Execute** or press **F5** to run the SQL script.
13. Review the **Messages** and **Data Output** panels for errors.
14. Refresh the database tree and confirm that the expected schemas, tables, and extensions are present.

> **Important:** Do not select **Restore** for a plain `.sql` file. pgAdmin's **Restore** option is intended for PostgreSQL archive files such as `.backup` or `.tar`. Use **Query Tool** for `base_imis.sql`.

### 6.6 GeoServer

Run command:

```bash
ansible-playbook playbooks/06_install_hardened_geoserver.yml
```

### What `06_install_hardened_geoserver.yml` does

Starts GeoServer locally in Docker, connects it to PostgreSQL, installs extensions, applies resource limits, configures credentials, and verifies the endpoint.

#### GeoServer prompts, values, and purpose

| Prompt                                          | Example/default value                     | Purpose                                                        |
| ----------------------------------------------- | ----------------------------------------- | -------------------------------------------------------------- |
| GeoServer install directory                     | `/home/imis_user/geoserver`               | Selects the host directory for GeoServer data and build files. |
| Create default printing `config.yml`?           | `yes`                                     | Creates the MapFish Print configuration when absent.           |
| A4 printing background PDF                      | Required when printing is enabled         | Supplies the A4 print background template.                     |
| A3 printing background PDF                      | Required when printing is enabled         | Supplies the A3 print background template.                     |
| GeoServer version                               | `2.28.2`                                  | Selects the GeoServer release.                                 |
| Container name                                  | `geo`                                     | Names the GeoServer Docker container.                          |
| Docker image name                               | Blank builds `custom-geoserver:<version>` | Selects an existing image or generated custom image.           |
| Localhost host port                             | `18082`                                   | Maps the GeoServer container port to the host.                 |
| Nginx Proxy Manager container                   | `nginx-proxy-manager`                     | Selects the NPM container for network integration.             |
| Primary GeoServer workspace                     | Optional                                  | Sets the main workspace used by the deployment.                |
| Authenticated WMS/WFS test layer                | Optional `workspace:layer`                | Selects a layer for service verification.                      |
| Skip demo data on a new data directory?         | `yes`                                     | Controls whether GeoServer demo data is omitted.               |
| Install stable extensions at startup?           | `yes`                                     | Enables installation of the selected extensions.               |
| Stable extensions                               | `authkey,css,printing,querylayer`         | Selects GeoServer extensions to install.                       |
| Remove `gt-complex*.jar` libraries?             | `no`                                      | Controls removal of unwanted GeoTools libraries.               |
| Unprivileged GeoServer UID/GID                  | `1002` / `1002`                           | Sets the runtime user and group ownership.                     |
| Set administrator credentials from environment? | `yes`                                     | Controls automated administrator credential setup.             |
| GeoServer administrator username                | `admin`                                   | Sets the GeoServer administrator login.                        |
| GeoServer administrator password                | Required and confirmed                    | Sets the GeoServer administrator credential.                   |
| Automate WMS/WFS/AuthKey security setup?        | `yes`                                     | Creates the service security configuration.                    |
| GeoServer OGC service username                  | `geoserver_service_user`                  | Names the OGC service account.                                 |
| GeoServer OGC service password                  | Required when enabled                     | Authenticates the OGC service account.                         |
| GeoServer OGC service role                      | `ROLE_GEOSERVER_SERVICE`                  | Assigns the service account role.                              |
| PostgreSQL Docker network                       | `postgres14_backend`                      | Connects GeoServer to PostgreSQL.                              |
| Container memory limit/reservation              | `2g` / `512m`                             | Controls GeoServer container memory allocation.                |
| JVM heap maximum                                | `1536m`                                   | Limits Java heap usage.                                        |
| Container CPU limit                             | `2.0`                                     | Limits GeoServer CPU allocation.                               |
| Back up existing GeoServer data?                | `yes`                                     | Protects existing data before deployment.                      |
| Pause at local Nginx checkpoint?                | `yes`                                     | Adds a checkpoint before Nginx-related steps.                  |

Current mapping:

```text
Container: geoserver
Server:    127.0.0.1:18082
Container: 8080
NPM config: /home/imis_user/geoserver/NPM_ADVANCED_CONFIG.conf
```

### Open GeoServer directly

1. On Windows, press **Win + R**, type `cmd`, and press **Enter** to open Command Prompt.
2. Paste the following command. Replace `<server_name>` with your SSH username and `<server_ip>` with the server IP address:

```cmd
ssh -p 2224 -N -L 80:127.0.0.1:18082 <server_name>@<server_ip>
```

3. Press **Enter** and leave this Command Prompt window open.
4. Open a web browser and enter this URL in the address bar:

```text
http://localhost/geoserver
```

### Open GeoServer through Nginx Proxy Manager

1. Open a new Windows Command Prompt window by pressing **Win + R**, typing `cmd`, and pressing **Enter**.
2. Paste the following command. Replace `<server_name>` with your SSH username and `<server_ip>` with the server IP address:

```cmd
ssh -p 2224 -N -L 8080:127.0.0.1:80 <server_name>@<server_ip>
```

3. Press **Enter** and leave this Command Prompt window open. The tunnel is active while this window is running.
4. Open a web browser, click the address bar, enter the following URL, and press **Enter**:

```text
http://localhost:8080/geoserver
```

The `18082` tunnel accesses GeoServer directly. The local `8080` tunnel forwards to the server's Nginx Proxy Manager HTTP listener, which then proxies to the GeoServer container at `geoserver:8080`.

### Get the GeoServer AuthKey

The deployment creates the AuthKey UUID for the service user `geoserver_service_user`. Read it on the server with:

```bash
sudo sed -n '/<user[^>]*name="geoserver_service_user"/,/<\/user>/p' \
  /home/imis_user/geoserver/geoserver_data/security/usergroup/default/users.xml \
  | grep -oP '(?<=<property name="UUID">)[0-9a-fA-F-]{36}'
```

Use the returned UUID as the `authkey` URL parameter. Keep it secret and do not put it in source control, browser history, proxy logs, or application logs.

To locate the AuthKey user from the GeoServer web console:

1. Sign in at `http://localhost/geoserver` with the GeoServer administrator account.
2. Open **Security -> Users, Groups, and Roles**.
3. Open the **Users** tab and select `geoserver_service_user`.
4. The generated UUID/AuthKey is stored in the server file shown above; run the command above to read it.

The web console identifies the AuthKey user, while the generated UUID is read from `users.xml` on the server.

### Load the GeoServer workspace from the server

Run the workspace importer from the server. It creates a Python virtual environment, installs the required package, and starts the interactive workspace import process:

Run commands:

```bash
sudo apt update
```

```bash
sudo apt install -y python3-pip python3.12-venv
```

```bash
cd /home/imis_user/server-setup-ansible/files/geoserver_workspace/workspace_manifest_tool
```

```bash
chmod +x run_workspace_manifest_tool.sh
```

```bash
./run_workspace_manifest_tool.sh
```

Follow the prompts to select the workspace manifest and enter the destination GeoServer and PostgreSQL connection details. The target database, user, schema, and required PostGIS extensions must already exist before importing the workspace.

The workspace importer defaults the target GeoServer URL to `http://localhost/geoserver`. Keep this SSH tunnel running in another terminal while creating the workspace, layers, and styles:

```bash
ssh -p 2224 -N -L 80:127.0.0.1:18082 <admin_user>@<server_ip>
```

The tunnel must remain open for the importer to communicate with GeoServer.

### Apply GeoServer `.env`-only changes

If only `/home/imis_user/geoserver/.env` was changed, recreate the GeoServer
service so Docker Compose reads the updated environment values. A rebuild is
not required for environment-only changes:

Run command:

```bash
cd /home/imis_user/geoserver
```

```bash
sudo docker compose up -d --force-recreate --no-build geoserver
```

```bash
sudo docker compose ps geoserver
```

Use `--build` only when the Dockerfile or image contents also changed.

#### GeoServer CORS prompts, values, and purpose

| Prompt                           | Example/default value         | Purpose                                                |
| -------------------------------- | ----------------------------- | ------------------------------------------------------ |
| GeoServer container name         | `geoserver` or current `geo`  | Selects the container whose CORS settings are changed. |
| Host path to `geoserver-web.xml` | Blank auto-detects the mount  | Locates the GeoServer CORS configuration file.         |
| CORS origins                     | Required, comma-separated     | Lists browser origins allowed by GeoServer.            |
| Wildcard confirmation            | `allow wildcard` only for `*` | Explicitly confirms allowing every origin.             |
| Apply CORS update and restart?   | `yes`                         | Writes the change and restarts GeoServer.              |

### Configure GeoServer in Nginx Proxy Manager

#### Open Nginx Proxy Manager

1. On Windows, press **Win + R**, type `cmd`, and press **Enter**.
2. Run the following command, replacing `<server_name>` and `<server_ip>` with your SSH username and server IP address:

```cmd
ssh -p 2224 -N -L 8181:127.0.0.1:81 <server_name>@<server_ip>
```

3. Press **Enter** and keep this Command Prompt window open.
4. Open a browser and go to `http://localhost:8181`.
5. Sign in to the Nginx Proxy Manager administrator page.

#### Add the GeoServer proxy host

In Nginx Proxy Manager, select **Hosts -> Proxy Hosts -> Add Proxy Host** and use these values:

| Setting          | Value       |
| ---------------- | ----------- |
| Scheme           | 'http'      |
| Forward Hostname | 'geoserver' |
| Forward Port     | '8080'      |

For **Domain Names**, enter `localhost` for local testing. Do not use `127.0.0.1:18082` as the Nginx Proxy Manager upstream. Port `18082` is only for direct verification and SSH tunneling.

#### Paste the Advanced configuration

1. On the server, display the generated configuration:

```bash
sudo cat /home/imis_user/geoserver/NPM_ADVANCED_CONFIG.conf
```

2. Copy the complete output, starting with `location = /geoserver` and ending with the final GeoServer `location` block.
3. In the Nginx Proxy Manager **Add Proxy Host** page, open the **Advanced** tab.
4. Paste the configuration into the **Custom Nginx Configuration** field.
5. Save the proxy host and wait for Nginx Proxy Manager to reload successfully.

The Advanced configuration redirects `/geoserver` to `/geoserver/`, forwards GeoServer web, REST, login, logout, WMS, WFS, WCS, WMTS, and OWS requests to `geoserver:8080`, and blocks unsupported GeoServer paths. Paste the generated file exactly; do not add an `http {}` or `server {}` wrapper.

After saving the proxy host, use the Nginx Proxy Manager tunnel described above and open `http://localhost:8080/geoserver` in the browser.

## 7. Shared Nginx Initial Setup

Return to the Ansible repository directory:

```bash
cd /home/imis_user/server-setup-ansible
```

Run command:

```bash
ansible-playbook playbooks/07_shared_nginx_register.yml
```

### What `07_shared_nginx_register.yml` does

Creates the shared Docker networks, shared Nginx directories, shared configuration, and shared-nginx container. Run it before Laravel deployment.

This playbook has no interactive prompts. Its configured values are the shared base path, shared Nginx directory and container, Nginx Proxy Manager container, and the `shared-web` and `npm-network` Docker networks.

### 6.8 Laravel Application

Run command:

```bash
ansible-playbook playbooks/08_Laravel_webapp_deploy.yml
```

### What `08_Laravel_webapp_deploy.yml` does

Clones the Laravel repository, creates application containers, writes the .env file, configures PostgreSQL and GeoServer integration, runs Composer and optional NPM setup, starts the application, creates the project Nginx configuration, and reloads shared Nginx.

#### Laravel deployment prompts and example values

The following table records the prompts and values shown in the example deployment run. Pressing **Enter** accepts the displayed default; sensitive values such as passwords and API keys should not be written in deployment notes.

| Prompt                                                  | Example value                                    | Purpose                                                                                    |
| ------------------------------------------------------- | ------------------------------------------------ | ------------------------------------------------------------------------------------------ |
| Project name                                            | `base-imis`                                      | Names the application project, directory, Compose project, and containers.                 |
| Laravel project repo URL                                | `https://github.com/base-imis/web_app.git`       | Identifies the Laravel repository to clone.                                                |
| Web app folder name                                     | Blank; uses the default                          | Sets the application folder name when it must differ from the project-derived name.        |
| Git branch                                              | `main`                                           | Selects the source-code branch to deploy.                                                  |
| Domain/subdomain                                        | `<domian_name>`                                  | Sets the application hostname used by shared Nginx and Laravel `APP_URL`.                  |
| Use database?                                           | `y`                                              | Enables PostgreSQL configuration and connectivity checks.                                  |
| Use GeoServer?                                          | `y`                                              | Enables GeoServer network and environment configuration.                                   |
| Configure the PII encryption feature?                   | `y`                                              | Enables PII key setup, migrations, permissions, and encryption processing.                 |
| Is the database completely empty?                       | Blank; uses `n`                                  | Treats the database as existing, so empty-database migrations are not selected.            |
| Run npm setup?                                          | `y`                                              | Installs frontend dependencies and runs the available production build.                    |
| Base path                                               | Blank; uses `/home/imis_user`                    | Sets the parent directory for the project and Docker files.                                |
| Docker template repository                              | `https://github.com/base-imis/docker_config.git` | Identifies the repository containing the Docker and Compose template.                      |
| Docker template Git branch                              | Blank; uses `main`                               | Selects the Docker template branch.                                                        |
| Proceed if the project directory already exists?        | Blank; uses the default                          | Controls whether an existing project directory may be reused.                              |
| Database type                                           | Blank; uses `local`                              | Selects local Docker-network database connectivity rather than remote database mode.       |
| PostgreSQL network name                                 | Blank; uses `postgres14_backend`                 | Connects the application containers to PostgreSQL through Docker DNS.                      |
| Shared web Docker network name                          | Blank; uses `shared-web`                         | Connects the application to the shared Nginx network.                                      |
| Verify Laravel-owned clickjacking headers?              | Blank; uses `yes`                                | Checks that shared Nginx preserves the Laravel security headers.                           |
| Optional public URL for final clickjacking verification | Blank; skipped                                   | Provides a public URL for an external final header check when available.                   |
| PHP major.minor.patch version                           | Blank; uses `8.2.35`                             | Sets the PHP version for PHP, Artisan, and Composer containers.                            |
| PostgreSQL container name or database host              | Blank; uses `postgres14`                         | Tells Laravel which PostgreSQL host or container to use.                                   |
| Database port                                           | Blank; uses `5432`                               | Sets the PostgreSQL port used inside the Docker network.                                   |
| Database name                                           | `defaultdb`                                      | Selects the PostgreSQL database used by Laravel.                                           |
| Database username                                       | `imis_user`                                      | Selects the PostgreSQL role used by Laravel.                                               |
| Database user password                                  | Blank in the captured input                      | Supplies the password for the Laravel PostgreSQL user; enter and confirm it when required. |
| Confirm database user password                          | Blank in the captured input                      | Confirms the Laravel PostgreSQL user password.                                             |
| GeoServer workspace name                                | `base_imis`                                      | Selects the GeoServer workspace used by the application.                                   |
| GeoServer AuthKey secret                                | Blank; skipped                                   | Supplies an optional secret for authenticated GeoServer requests.                          |
| Google API key                                          | Blank; skipped                                   | Enables Google-backed application features when a key is supplied.                         |
| App name                                                | `IMIS`                                           | Sets the Laravel application name in `.env`.                                               |
| App environment                                         | Blank; uses `production`                         | Selects production Laravel behavior and configuration.                                     |
| App debug                                               | `true`                                           | Enables Laravel debug behavior; use `false` for a production deployment.                   |
| Additional environment entries                          | Blank; skipped                                   | Adds extra `.env` values in `KEY=value;KEY2=value2` format.                                |
| `SEED_SUPER_ADMIN_NAME`                                 | Blank; skipped                                   | Sets the optional seeded administrator’s display name.                                     |
| `SEED_SUPER_ADMIN_USERNAME`                             | Blank; skipped                                   | Sets the optional seeded administrator’s login username.                                   |
| `SEED_SUPER_ADMIN_EMAIL`                                | Blank; skipped                                   | Sets the optional seeded administrator’s email address.                                    |
| `SEED_SUPER_ADMIN_PASSWORD`                             | Blank; skipped                                   | Sets and confirms the optional seeded administrator’s password.                            |

Example project configuration:

```text
/home/imis_user/shared-nginx/nginx/conf.d/myapp.conf
```

Check that the PostgreSQL, shared web, and GeoServer Docker networks exist:

```bash
sudo docker network ls
```

```bash
sudo docker network inspect postgres14_backend shared-web geoserver_proxy
```
