+++
title = "Guacamole: Hugo's favorite side-dish"
description = 'Deploying a web-based shell via Guacamole with RDP for remote graphical access'
date = 2026-03-24T17:50:24-04:00
tags= ["linux", "servers", "tech", "web"]
draft = true
+++

While I tend to prefer CLI and TUI's for most of my needs (especially on a server), it is sometimes helpful to have a remote graphical machine, accessible from anywhere. I've tried a few commercial services but they are both pricey and limited in features. Since these services utilize standard protocols, I decided to implement my own solution allowing me to browse the web and use discord securely from any machine with a web browser. It also serves as a perfect complement to my webserver when debugging over ssh is not ideal.

## Background Services

In order to get this setup working we need to install and configure a few components for a graphical desktop environment:

- A Desktop Environment: I chose to use [XFCE](https://www.xfce.org/) to keep things light and performant
- [XRDP](https://www.xrdp.org/): To serve the display over Remote Desktop Protocol to the frontend
- [Docker](https://www.docker.com/): To simplofy services and sandbox them from the main machine
- [Gucamole](https://guacamole.apache.org/): To provide a web portal we can serve from the web server

### Desktop Installation

First we need to install the base packages for XFCE. We do not want to install any type of display manager, this will work headless and be launched by xrdp:

```bash
sudo apt update
sudo apt install xfce4 xfce4-goodies -y
```

Next we will install XRDP, which will serve as the remote streaming server to provide the connection to the Guacamole frontend:

```bash
sudo apt install xrdp -y
```

We will then configure XFCE when we start a session over RDP by creating `~/.xsession` and adding the following line:

```bash
startxfce4
```

Now we will configure the XRDP server to start and run in the background automatically, and to work properly with SSL certificates for secure access

```bash
sudo systemctl enable --now xrdp
sudo adduser xrdp ssl-cert
sudo systemctl restart xrdp
```

At this point we can now test our RDP session's connectivity by attempting to connect to our server using an XRDP client. MAke sure you are allowing traffic to the RDP port through your firewall for the local client's IP when doing this, it should be locked down to HTTP and HTTPS only once setup is complete.

### Service Installation

Docker installation is fairly straightforward and should be done in accordance with the latest [official documentation](https://docs.docker.com/engine/install/ubuntu/):

```bash
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update

sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Once we have docker installed and running we will install the Guacamole docker image in a few steps:

1. Initialize and pull a base sql database image from the container once to import to the container:

    ```bash
    docker run --rm guacamole/guacamole:latest /opt/guacamole/bin/initdb.sh --mysql > initdb.sql
    ```

2. Create a base `docker-compose.yml` file from which we will build the Guacamole component containers. To start, add the following contents:

    ```yaml
    version: '3'
    services:
        guacdb:
            container_name: guacamoledb
            image: mariadb:10.9.5
            network_mode: "host"
            restart: unless-stopped
            environment:
                MYSQL_ROOT_PASSWORD: "CREATE_A_ROOT_PASSWORD_HERE"
                MYSQL_DATABASE: 'guacamole_db'
                MYSQL_USER: 'guacamole_user'
                MYSQL_PASSWORD: "CREATE_A_DATABASE_PASSWORD_HERE"
            volumes:
                - './db-data:/var/lib/mysql'
    volumes:
        db-data:
    ```

4. Start the container using `docker compose up -d` and copy the generated sql DB into the container:

    ```bash
    docker cp initdb.sql guacamoledb:/initdb.sql
    ```

5. Input the DB contents into the server, first start an exec session in the container:

    ```bash
    docker exec -it guacamoledb bash
    ```

    And from within the container exec session, concatenate the DB contents into `guacamole_db`:

    ```bash
    cat /initdb.sql | mysql -u root -p guacamole_db
    exit
    ```

6. Stop the docker container so we can finishing configuring the rest of the components:

    ```bash
    docker compose down
    ```

7. Add the following services to the `docker-compose.yml` file after `guacdb:`:

    ```yaml
    guacd:
        container_name: guacd
        image: guacamole/guacd:latest
        network_mode: "host"
        restart: unless-stopped
    guacamole:
        container_name: guacamole
        image: guacamole/guacamole:latest
        network_mode: "host"
        restart: unless-stopped
        environment:
            GUACD_HOSTNAME: "127.0.0.1"
            MYSQL_HOSTNAME: "127.0.0.1"
            MYSQL_DATABASE: "guacamole_db"
            MYSQL_USER: "guacamole_user"
            MYSQL_PASSWORD: "DATABASE_PASSWORD_HERE"
            TOTP_ENABLED: "true"    # Only if you want MFA
        depends_on:
            - guacdb
            - guacd
    ```

8. We can now bring the containers back online to start the database, daemon, and frontend app:

    ```bash
    docker compose up -d
    ```

9. At this point you should access the frontend via ip and port to ensure it is running, log in to the admin account (`guacadmin/guacadmin`) and change the password.

### Web Server Setup
