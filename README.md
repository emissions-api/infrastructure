# Infrastructure

This repository contains the files and descriptions to setup the infrastructure for [Emissions API](https://emissions-api.org/).

## Manual Setup

This is a manual setup guide for our test setup on `demo.emissions-api.org`.
As most of the setup guides this guide is probably not completely and at least slightly outdated.

### Prerequisites

This setup is based on [Debian 10](https://www.debian.org/index.de.html), aka. *Buster*.
You will need root access to the machine.
Best would be a user with root permission via `sudo`, since later steps with the python virtual environment might have problems when running as root.

### Install System Packages

To install the required package just run

```
sudo apt-get update
sudo apt-get install nginx postgresql postgis git python python3-gdal \
    python3-psycopg2 python3-pycurl python3-numpy python3-venv python3-wheel
```

### Configure Database

The database needs to be setup.
Become the user `postgres`, create a user and a database and enable the [PostGIS](https://postgis.net/) extension on the database.
You need to set a proper password for the `emissionsapi` user which is later be used in the configuration file.

```
sudo su - postgres
createuser --pwprompt emissionsapi
createdb --owner emissionsapi emissionsapi
echo "CREATE EXTENSION postgis;
CREATE EXTENSION postgis_topology;
CREATE EXTENSION postgis_sfcgal;
CREATE EXTENSION fuzzystrmatch;
CREATE EXTENSION address_standardizer;
CREATE EXTENSION address_standardizer_data_us;
CREATE EXTENSION postgis_tiger_geocoder;
" | psql emissionsapi
```

### Install Emissions API Software

The Emissions API software is installed in a virtual environment.
So simply create one.

```
sudo python3 -m venv --system-site-packages /opt/emissions-api-env
```

After that clone the Emissions API repository and install its requirements plus [gunicorn](https://gunicorn.org/)

```
sudo git clone https://github.com/emissions-api/emissions-api.git /opt/emissions-api
sudo /opt/emissions-api-env/bin/pip install -r /opt/emissions-api/requirements.txt
sudo /opt/emissions-api-env/bin/pip install gunicorn
```

We need a proper configuration file for the service, so create a file `/etc/emissionsapi.yml` with the following content

```
storage: /opt/data
download:
  date:
    begin: '2019-02-1T00:00:00.000Z'
    end: '2019-03-01T00:00:00.000Z'
  country:
    - DE
```

`<password>` has to be replaced with the password set during the database setup.
With this configuration all data sets for the February 2019 containing data points within Germany are downloaded.

To run the Emissions API web service on boot we have to create a [systemd](https://www.freedesktop.org/wiki/Software/systemd/) service file under `/etc/systemd/system/emissionsapi-web.service` with the following content

```
[Unit]
Description=Emissions API Web Service

[Service]
ExecStart=/opt/emissions-api-env/bin/gunicorn --workers 4 emissionsapi.web:app
WorkingDirectory=/opt/emissions-api/
RestartSec=5
Restart=always

[Install]
WantedBy=multi-user.target
```

This service file has to be enabled and started.

```
sudo systemctl enable emissionsapi-web.service
sudo systemctl start emissionsapi-web.service
```

### Setup NGINX

Last but not least we have to setup the reverse proxy [NGINX](https://www.nginx.com/).

First we delete the symbolic link to the default website NGINX delivers under Debian

```
sudo rm /etc/nginx/sites-enabled/default
```

After that we create a new server configuration file `/etc/nginx/sites-available/demo.emissions-api.org.conf` with the following content

```
server {
    listen 80;
    listen [::]:80;
    server_name demo.emissions-api.org;
    location / {
        proxy_pass http://127.0.0.1:8000;
    }
}
```
After that we create a new symbolic link and reload the NGINX service

```
sudo ln -s /etc/nginx/sites-available/demo.emissions-api.org.conf /etc/nginx/sites-enabled/
sudo systemctl restart nginx
```
