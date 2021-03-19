# Development guide

  * [Installation](#installation)
  * [Upgrade](#upgrade)

## Installation

This guide is not intended to be used for production environments. Please use [this guide](production.md) for production.

### 1) Dependencies

1. On a fresh Debian/Ubuntu, as root user, install basic utility programs needed for the installation

```
# apt-get install curl sudo unzip vim
```

2. It would be wise to disable root access and to continue this tutorial with a user with sudoers group access

3. Install NodeJS 12.x:
[https://nodejs.org/en/download/package-manager/#debian-and-ubuntu-based-linux-distributions](https://nodejs.org/en/download/package-manager/#debian-and-ubuntu-based-linux-distributions)
4. Install yarn, and be sure to have [a recent version](https://github.com/yarnpkg/yarn/releases/latest):
[https://yarnpkg.com/en/docs/install#linux-tab](https://yarnpkg.com/en/docs/install#linux-tab)

5. Run:

```
sudo apt update
sudo apt install nginx ffmpeg postgresql postgresql-contrib openssl g++ make redis-server git python-dev cron wget
ffmpeg -version # Should be >= 4.1
g++ -v # Should be >= 5.x
```

Now that dependencies are installed, before running PeerTube you should start PostgreSQL and Redis:

```
sudo systemctl start redis postgresql
```

### 2) PeerTube user

Create a `peertube` user with `/var/www/peertube` home:

```
$ sudo useradd -m -d /var/www/peertube -s /bin/bash -p peertube peertube
```

Set its password:
```
$ sudo passwd peertube
```

### 3) Database

Create the production database and a peertube user inside PostgreSQL:

```
$ sudo -u postgres createuser -P peertube
```

Here you should enter a password for PostgreSQL `peertube` user, that should be copied in `production.yaml` file.
Don't just hit enter else it will be empty.

```
$ sudo -u postgres createdb -O peertube -E UTF8 -T template0 peertube_dev
```

Then enable extensions PeerTube needs:

```
$ sudo -u postgres psql -c "CREATE EXTENSION pg_trgm;" peertube_dev
$ sudo -u postgres psql -c "CREATE EXTENSION unaccent;" peertube_dev
```

### 4) Prepare PeerTube directory

Fetch the latest tagged version of Peertube
```
$ VERSION=$(curl -s https://api.github.com/repos/chocobozzz/peertube/releases/latest | grep tag_name | cut -d '"' -f 4) && echo "Latest Peertube version is $VERSION"
```

Open the peertube directory, create a few required directories
```
$ cd /var/www/peertube
$ sudo -u peertube mkdir config storage versions
```

Download the latest version of the Peertube client, unzip it and remove the zip
```
$ cd /var/www/peertube/versions
$ sudo -u peertube wget -q "https://github.com/Chocobozzz/PeerTube/releases/download/${VERSION}/peertube-${VERSION}.zip"
$ sudo -u peertube unzip -q peertube-${VERSION}.zip && sudo -u peertube rm peertube-${VERSION}.zip
```

Install Peertube:
```
$ cd /var/www/peertube
$ sudo -u peertube ln -s versions/peertube-${VERSION} ./peertube-latest
$ cd ./peertube-latest && sudo -H -u peertube yarn install --production --pure-lockfile
```

### 5) PeerTube configuration

Copy the default configuration file that contains the default configuration provided by PeerTube.
You **must not** update this file.

```
$ cd /var/www/peertube
$ sudo -u peertube cp peertube-latest/config/default.yaml config/default.yaml
```

Now copy the development example configuration (same as production without certificate) :

```
$ cd /var/www/peertube
$ sudo -u peertube cp peertube-latest/config/production.yaml.example config/development.yaml
```

Then edit the `config/development.yaml` file according to your webserver
and database configuration (`webserver`, `database`, `redis`, `smtp` and `admin.email` sections in particular).
Keys defined in `config/development.yaml` will override keys defined in `config/default.yaml`.

**PeerTube does not support webserver host change**. Even though [PeerTube CLI can help you to switch hostname](https://docs.joinpeertube.org/maintain-tools?id=update-hostjs) there's no official support for that since it is a risky operation that might result in unforeseen errors.

### 6) Webserver

We only provide official configuration files for Nginx.

Copy the nginx configuration template:

```
$ sudo cp /var/www/peertube/peertube-latest/support/nginx/peertube-nocert /etc/nginx/sites-available/peertube
```

Then set the domain for the webserver configuration file.
Replace `[peertube-domain]` with the domain for the peertube server.

```
$ sudo sed -i 's/${WEBSERVER_HOST}/[peertube-domain]/g' /etc/nginx/sites-available/peertube
$ sudo sed -i 's/${PEERTUBE_HOST}/127.0.0.1:9000/g' /etc/nginx/sites-available/peertube
```

Then modify the webserver configuration file. Please pay attention to the `alias` keys of the static locations.
It should correspond to the paths of your storage directories (set in the configuration file inside the `storage` key).

```
$ sudo vim /etc/nginx/sites-available/peertube
```

Activate the configuration file:

```
$ sudo ln -s /etc/nginx/sites-available/peertube /etc/nginx/sites-enabled/peertube
```

```
systemctl restart nginx
```


### 7) TCP/IP Tuning

**On Linux**

```
$ sudo cp /var/www/peertube/peertube-latest/support/sysctl.d/30-peertube-tcp.conf /etc/sysctl.d/
$ sudo sysctl -p /etc/sysctl.d/30-peertube-tcp.conf
```

Your distro may enable this by default, but at least Debian 9 does not, and the default FIFO
scheduler is quite prone to "Buffer Bloat" and extreme latency when dealing with slower client
links as we often encounter in a video server.

### 8) make PeerTube a service 

**systemd**

If your OS uses systemd, copy the configuration template:

```
$ sudo cp /var/www/peertube/peertube-latest/support/systemd/peertube.service /etc/systemd/system/
```

Check the service file (PeerTube paths and security directives):

```
$ sudo vim /etc/systemd/system/peertube.service
```


Tell systemd to reload its config:

```
$ sudo systemctl daemon-reload
```

If you want to start PeerTube on boot:

```
$ sudo systemctl enable peertube
```

Run:

```
$ sudo systemctl start peertube
$ sudo journalctl -feu peertube
```

Run and print last logs:

```
$ sudo /etc/init.d/peertube start
$ tail -f /var/log/peertube/peertube.log
```

### Administrator

The administrator password is automatically generated and can be found in the PeerTube
logs (path defined in `production.yaml`). You can also set another password with:

```
$ cd /var/www/peertube/peertube-latest && NODE_CONFIG_DIR=/var/www/peertube/config NODE_ENV=production npm run reset-password -- -u root
```

Alternatively you can set the environment variable `PT_INITIAL_ROOT_PASSWORD`,
to your own administrator password, although it must be 6 characters or more.

### What now?

Now your instance is up you can:

 * Add your instance to the public PeerTube instances index if you want to: https://instances.joinpeertube.org/
 * Check [available CLI tools](/support/doc/tools.md)

## Upgrade

### PeerTube instance

**Check the changelog (in particular BREAKING CHANGES!):** https://github.com/Chocobozzz/PeerTube/blob/develop/CHANGELOG.md

#### Auto

The password it asks is PeerTube's database user password.

```
$ cd /var/www/peertube/peertube-latest/scripts && sudo -H -u peertube ./upgrade.sh
```

#### Manually

Make a SQL backup

```
$ SQL_BACKUP_PATH="backup/sql-peertube_dev-$(date -Im).bak" && \
    cd /var/www/peertube && sudo -u peertube mkdir -p backup && \
    sudo -u postgres pg_dump -F c peertube_dev | sudo -u peertube tee "$SQL_BACKUP_PATH" >/dev/null
```

Fetch the latest tagged version of Peertube:

```
$ VERSION=$(curl -s https://api.github.com/repos/chocobozzz/peertube/releases/latest | grep tag_name | cut -d '"' -f 4) && echo "Latest Peertube version is $VERSION"
```

Download the new version and unzip it:

```
$ cd /var/www/peertube/versions && \
    sudo -u peertube wget -q "https://github.com/Chocobozzz/PeerTube/releases/download/${VERSION}/peertube-${VERSION}.zip" && \
    sudo -u peertube unzip -o peertube-${VERSION}.zip && \
    sudo -u peertube rm peertube-${VERSION}.zip
```

Install node dependencies:

```
$ cd /var/www/peertube/versions/peertube-${VERSION} && \
    sudo -H -u peertube yarn install --production --pure-lockfile
```

Copy new configuration defaults values and update your configuration file:

```
$ sudo -u peertube cp /var/www/peertube/versions/peertube-${VERSION}/config/default.yaml /var/www/peertube/config/default.yaml
$ diff /var/www/peertube/versions/peertube-${VERSION}/config/production.yaml.example /var/www/peertube/config/production.yaml
```

Change the link to point to the latest version:

```
$ cd /var/www/peertube && \
    sudo unlink ./peertube-latest && \
    sudo -u peertube ln -s versions/peertube-${VERSION} ./peertube-latest
```

### nginx

Check changes in nginx configuration:

```
$ cd /var/www/peertube/versions
$ diff "$(ls --sort=t | head -2 | tail -1)/support/nginx/peertube" "$(ls --sort=t | head -1)/support/nginx/peertube"
```

### systemd

Check changes in systemd configuration:

```
$ cd /var/www/peertube/versions
$ diff "$(ls --sort=t | head -2 | tail -1)/support/systemd/peertube.service" "$(ls --sort=t | head -1)/support/systemd/peertube.service"
```

### Restart PeerTube

If you changed your nginx configuration:

```
$ sudo systemctl reload nginx
```

If you changed your systemd configuration:

```
$ sudo systemctl daemon-reload
```

Restart PeerTube and check the logs:

```
$ sudo systemctl restart peertube && sudo journalctl -fu peertube
```

### Things went wrong?

Change `peertube-latest` destination to the previous version and restore your SQL backup:

```
$ OLD_VERSION="v0.42.42" && SQL_BACKUP_PATH="backup/sql-peertube_prod-2018-01-19T10:18+01:00.bak" && \
    cd /var/www/peertube && sudo -u peertube unlink ./peertube-latest && \
    sudo -u peertube ln -s "versions/peertube-$OLD_VERSION" peertube-latest && \
    sudo -u postgres pg_restore -c -C -d postgres "$SQL_BACKUP_PATH" && \
    sudo systemctl restart peertube
```
