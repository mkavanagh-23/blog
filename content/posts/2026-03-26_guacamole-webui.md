+++
title = "Remote GUI Access with Guacamole"
description = 'Deploying a web-based shell via Guacamole with RDP for remote graphical access'
date = 2026-03-26T17:50:24-04:00
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

Now that our services are installed and successfully running, we can set up a web server to secure our environment and provide our services publicly over DNS. I prefer using NGINX for it's ease of configuration and extensibility.

1. Install the NGINX server and ssl certificate management tools:

    ```bash
    sudo apt install nginx -y
    sudo systemctl enable --now nginx
    sudo apt install certbot python3-certbot-nginx -y
    ```

2. If you wish to setup basic http authentication for added security (you definitely should), we need to do the following:

    ```bash
    sudo apt install apache2-utils -y
    sudo htpasswd -c /etc/nginx/.guac_htpasswd HTTPAUTH_USERNAME_HERE
    sudo chmod 640 /etc/nginx/.guac_htpasswd
    sudo chown root:www-data /etc/nginx/.guac_htpasswd
    ```

3. Now we can create a configuration for our site in `/etc/nginx/sites-available/guacamole`. If you already have other services on your domain, be sure to create a separate conf file for guacamole and use `subdomain.domain.com` in the following. You will also need to ensure that there is an A record on your DNS pointing the proper domain/subdomain to your server IP address.

    ```nginx
    server {
        listen 80;
        server_name domain.com;

        # Redirect all HTTP to HTTPS
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl;
        http2 on;
        server_name domain.com;

        # Temporary self-signed placeholders (Certbot will replace these)
        ssl_certificate /etc/ssl/certs/ssl-cert-snakeoil.pem;
        ssl_certificate_key /etc/ssl/private/ssl-cert-snakeoil.key;

        # Basic Auth
        auth_basic "Restricted Access";
        auth_basic_user_file /etc/nginx/.guac_htpasswd;

        location / {
            proxy_pass http://127.0.0.1:8080/guacamole/;
            proxy_buffering off;

            proxy_http_version 1.1;

            # WebSocket support (critical for Guacamole)
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";

            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
    ```

4. Next you can enable the site by linking the config to `sites-available`, then test and restart the webserver

    ```bash
    sudo ln -s /etc/nginx/sites-available/guacamole /etc/nginx/sites-enabled/
    sudo nginx -t
    sudo systemctl restart nginx
    ```

5. Now we can obtain and enroll our SSL certificates, this will set up autorenewal:

    ```bash
    sudo certbot --nginx -d subdomain.domain.com
    ```

6. As a last step you should harden your install with firewall rules to block connections to all ports other than 80 and 443. I prefer to use UFW as it makes things fairly simple. After installing run the following commands to configure your rules, substitute them for your preferred firewall application if desired:

    ```bash
    sudo ufw default deny incoming  # Deny incoming connections by default
    sudo ufw allow ‘Nginx Full’ # Allow http and https traffic
    sudo ufw allow from 127.0.0.1 to any port 3389 # Allow local access to RDP protocol for guacamole
    sudo ufw allow from 127.0.0.1 to any port 8080 # Allow local access to Guacamole for nginx
    ```

You should now be able to access your Guacamole instance by visiting your configured (sub)domain. You will be prompted to enter the httpauth username and password you set during setup, and then can log in to the Guacamole app with the username and password you reset earlier. You can now go to Guacamole settings to set up a new connection and bind it to your RDP server:

- Procotol: `RDP`
- Network:
    - Hostname: `127.0.0.1`
    - Port: `3389`
- Authentication
    - Username: `YOUR LINUX USER`
    - Password: `LINUX USER PASS`
    - Security mode: `RDP encryption`
    - `Ignore server certificate`
- Display (I like to limit this to reasonable limits):
    - Width: `1920`
    - Height: `1080`
    - Color depth: `Low color (16-bit)`

Save the connection, and now from the main Guacamole homepage you can access your machine by selecting the connection.
