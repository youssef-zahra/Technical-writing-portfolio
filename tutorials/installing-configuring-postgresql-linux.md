# Installing and Configuring PostgreSQL on Linux

This tutorial walks through installing PostgreSQL on a Debian-based Linux system, creating a database and user, and making the basic configuration changes most setups need before they are ready for real use. The commands below were run on Ubuntu, but the same steps apply with minor package-manager differences on any Debian-based distribution.

## What you will end up with

- PostgreSQL installed and running as a system service
- A dedicated database and database user, rather than relying on the default superuser
- Remote connections enabled, if your setup needs them
- A basic understanding of where PostgreSQL's configuration actually lives, so future changes are not guesswork

## Step 1: Install PostgreSQL

Update your package index and install PostgreSQL along with the contrib package, which adds a handful of useful extensions and utilities:

```bash
sudo apt update
sudo apt install postgresql postgresql-contrib
```

Once installed, PostgreSQL starts automatically and registers itself as a system service. Confirm it is running:

```bash
sudo systemctl status postgresql
```

You should see `active (running)` in the output. If it is not running, start it with `sudo systemctl start postgresql`, and enable it to start automatically on boot with `sudo systemctl enable postgresql`.

## Step 2: Access the default PostgreSQL user

The installation creates a Linux system user named `postgres`, which maps to a PostgreSQL superuser of the same name. Switch to that user to run initial administrative commands:

```bash
sudo -i -u postgres
psql
```

This drops you into the `psql` command-line interface, connected as the `postgres` superuser. You will see a prompt that looks like `postgres=#`.

## Step 3: Create a dedicated database and user

Running everything as the default superuser works for a quick test, but it is not how you want a real setup configured. Create a dedicated user and database instead, scoped to what you are actually building:

```sql
CREATE USER appuser WITH PASSWORD 'replace_with_a_real_password';
CREATE DATABASE appdb OWNER appuser;
GRANT ALL PRIVILEGES ON DATABASE appdb TO appuser;
```

Exit `psql` with `\q`, then exit back to your normal shell with `exit`.

## Step 4: Verify the new user can connect

Test the new user and database from your regular shell, not as the `postgres` system user:

```bash
psql -U appuser -d appdb -h localhost -W
```

The `-W` flag prompts for the password you just set. If this connects successfully and drops you into the `appdb=>` prompt, the user and database are working correctly.

If the connection is refused at this point, it is almost always because of the next step, PostgreSQL's default configuration does not allow password-based local connections until you tell it to.

## Step 5: Configure client authentication

PostgreSQL controls who is allowed to connect, and how, through a file called `pg_hba.conf` (host-based authentication). Find it:

```bash
sudo -u postgres psql -c "SHOW hba_file;"
```

This typically resolves to something like `/etc/postgresql/16/main/pg_hba.conf`. Open it with a text editor and look for the line governing local connections:

```
local   all             all                                     peer
```

`peer` authentication only works when your Linux system username matches your PostgreSQL username, which is why the connection in Step 4 may have failed. Change it to `md5` (password-based authentication) for the relevant line:

```
local   all             all                                     md5
```

Restart PostgreSQL for the change to take effect:

```bash
sudo systemctl restart postgresql
```

## Step 6: Enable remote connections, if you need them

By default, PostgreSQL only listens for connections from the local machine. If your setup needs to accept connections from elsewhere, two files need changes.

First, edit `postgresql.conf` (in the same directory as `pg_hba.conf`) and update the `listen_addresses` setting:

```
listen_addresses = '*'
```

Second, add a line to `pg_hba.conf` allowing connections from the network range you actually expect traffic from, rather than opening it to everything:

```
host    all             all             192.168.1.0/24          md5
```

Restart PostgreSQL again after making these changes. And treat this step carefully, opening PostgreSQL to remote connections without also configuring a firewall and strong authentication is a common way real deployments get compromised. This step is about making the service reachable, not about making it production-secure on its own.

## Why the configuration file locations matter

New PostgreSQL users often lose time editing the wrong file, or editing the right file but not realizing a restart is required for the change to apply. Knowing that `pg_hba.conf` controls who can connect and how, while `postgresql.conf` controls how the server itself behaves, and that both live in a version-specific directory under `/etc/postgresql/`, removes most of the guesswork from troubleshooting a connection problem later.


## The short version

Install the packages, confirm the service is running, create a dedicated database and user instead of using the default superuser, then adjust `pg_hba.conf` to allow the authentication method your setup actually needs. Most installation problems people run into after this point come down to one of two things: `pg_hba.conf` not permitting the connection method being used, or a configuration change made without a service restart afterward.
