# Deploy ASReview LAB Server with authentication (Docker Compose)

> ⚠️ Deploying Docker containers in a public environment requires careful
  consideration of security implications. Exposing services without proper
  safeguards can lead to potential security vulnerabilities, unauthorized
  access, and data breaches.

This repository, ASReview Server Stack, contains a recipe for building an
authenticated version of the ASReview LAB application in Docker containers with
Docker Compose. It allows multiple users to access the application and create
private projects. Users need to sign up (with various authentication methods)
and sign in to access the application.

> ℹ️ Looking for a standalone Dockerfile with ASReview LAB, simulate, insights or
  datatools? (Perfect for local use or small applications) See
  https://asreview.readthedocs.io/en/stable/installation.html#install-with-docker
  or https://github.com/asreview/asreview/pkgs/container/asreview.

## Prerequisites

A deployment requires a server: a Linux machine with administrator
(`root`/`sudo`) access. The following software must be installed on this server:

- **Docker** and the **Docker Compose** plugin. Docker runs each part of the
  application in an isolated unit called a Docker container; Docker Compose
  starts and connects these Docker containers together.
- **git**, used to download (clone) this repository onto the server.

Installation procedures for this software differ per operating system. Follow the
official instructions for [Docker Engine](https://docs.docker.com/engine/install/)
and the [Docker Compose plugin](https://docs.docker.com/compose/install/); git is
available through the package manager of every common Linux distribution. Once
this software is installed, the remaining deployment steps are identical across
operating systems, because Docker runs the application in the same way on each.

The two items above are sufficient for a basic deployment over plain HTTP. The
NGINX web server and the PostgreSQL database both run inside their own Docker
containers and are therefore installed automatically by Docker; they do not need
to be installed on the server separately.

Depending on the features required, the following are also needed:

- **A domain name**, with a DNS A record pointing to the server's IP address.
  This is required for HTTPS and optional otherwise. Without a domain name the
  application remains reachable by IP address over plain HTTP.
- **Certbot**, or another tool for obtaining TLS certificates. This is required
  only when the application is served over **HTTPS**. See [Upgrading security:
  migrate to HTTPS](#upgrading-security-migrate-to-https).
- **An SMTP service** such as [SendGrid](https://sendgrid.com/). This is
  required only for **email verification** and password-reset messages. See
  [Email server](#email-server).
- **Open firewall ports**, at minimum the port on which the application is
  served (for example, port `80` for HTTP, or ports `80` and `443` for HTTPS).

## Quick start (HTTP, without a domain name)

This section describes the quickest deployment: the application served over plain
HTTP and reached through the server's IP address, without a domain name or
certificates. Provided the [prerequisites](#prerequisites) are installed, the
steps below are identical on Ubuntu, Debian, CentOS, RHEL, and other common
distributions.

> ⚠️ Plain HTTP transmits all data, including passwords, unencrypted. This setup
  is suitable for testing or a trusted internal network. For any public-facing
  deployment, continue with [Upgrading security: migrate to
  HTTPS](#upgrading-security-migrate-to-https).

### 1. Get the code

Clone this repository onto the server and enter the created directory:

```
$ git clone https://github.com/asreview/asreview-server-stack.git
$ cd asreview-server-stack
```

All remaining commands are run from this directory.

### 2. Configure the environment

Three secret values are needed: a database password, a secret key, and a
password salt. On a system with OpenSSL installed, these can be generated as
follows:

```
$ openssl rand -base64 24   # database password
$ openssl rand -hex 32      # SECRET_KEY
$ openssl rand -hex 16      # SECURITY_PASSWORD_SALT
```

The configuration files described below are plain text and can be edited with any
text editor available on the server, for example `nano` (beginner-friendly) or
`vim`.

The `.env` file holds the deployment parameters. At a minimum, the PostgreSQL
password must be changed before deploying. The default frontend port is 8080;
setting it to 80 removes the port number from the application's URL.

```
FRONTEND_EXTERNAL_PORT=8080
BACKEND_EXTERNAL_PORT=8081

WORKERS=6

POSTGRES_PASSWORD="<the generated database password>"
POSTGRES_USER="postgres"
POSTGRES_DB="asreview_db"
```

See [Parameters in the .env file](#parameters-in-the-env-file) for a full
description of each parameter.

The `asreview_config.toml` file holds the application configuration. Before a
public deployment, the default `SECRET_KEY` and `SECURITY_PASSWORD_SALT` values
must be replaced with the generated `SECRET_KEY` and `SECURITY_PASSWORD_SALT`
values:

```
SECRET_KEY = "<the generated SECRET_KEY>"
SECURITY_PASSWORD_SALT = "<the generated SECURITY_PASSWORD_SALT>"
```

### 3. Build and start the containers

The following command builds the ASReview Docker image and starts all three
Docker containers in attached mode, so that the log output remains visible:

```
$ docker compose up
```

The first build takes a few minutes. The deployment is ready once the log output
settles and the backend container reports that it is serving requests.

### 4. Verify and run in the background

While the containers run, navigate in a browser to the server's address and the
configured frontend port, for example `http://<server-IP>:8080` (or
`http://<server-IP>` when the port is set to 80). The ASReview LAB sign-in page
should appear. By default, account creation is enabled, so the first user can
register directly from the sign-in page.

Ensure that the frontend port is open in the server's firewall. The firewall tool
differs per distribution (for example, `ufw` on Ubuntu and Debian, `firewalld` on
CentOS and RHEL).

Cloud providers such as Hetzner and DigitalOcean apply a **second, independent
firewall** in their web console, separate from the firewall on the server itself.
When the application is unreachable even though the server's own firewall allows
the port, this provider-level firewall is the most common cause: the port must be
opened there as well.

Once the application responds, stop the attached containers with `Ctrl+C` and
start them again in detached (background) mode:

```
$ docker compose up -d
```

The application then keeps running after the terminal is closed and restarts
automatically if the server reboots.

## Deploy to production

To run the ASReview LAB application as a shared service, a more elaborate Docker
container setup is advised. A common, robust setup for a Flask application like
ASReview LAB is to use [Gunicorn](https://gunicorn.org/) as a WSGI server and
use [NGINX](https://www.nginx.com/) as a reverse proxy. See the Flask
documentation on [Deploying to
Production](https://flask.palletsprojects.com/en/3.0.x/deploying/) for more
information.

ASReview Server Stack consists of three Docker containers: a PostgreSQL
database, an ASReview application, and an NGINX container. [Docker
Compose](https://docs.docker.com/compose/) is used to run and serve this
multi-container application. The following files in this folder are of interest:

- `.env` - An environment variable file for all relevant (secret) parameters
  (ports, frontend-domain, database, and Gunicorn related parameters)
- `asreview.conf` - an NGINX configuration file
- `docker-compose.yml` - the Docker Compose file that will create the Docker
  containers
- `asreview_config.toml` - the ASReview LAB config file with, for example,
  authentication options and email server configuration.

### Running the containers

A clone or copy of the ASReview Server Stack folder is required. From the
**root** folder of the application, the `docker compose` command starts the
Docker containers in attached mode:

```
$ docker compose up
```

### Email server

For account verification, but also for the forgot-password feature, an email
server is required. However, maintaining an email server can be demanding. This can be avoided by using a third-party service like
[SendGrid](https://sendgrid.com/). Email server
settings can be set in the `asreview_config.toml` file.

#### SendGrid

This recipe uses the SMTP Relay Service from
[SendGrid](https://sendgrid.com/): every email sent by the ASReview application
is relayed by this service. SendGrid is free as long as the application does not
send more than 100 emails per day. Receiving reply emails from end users is not
possible when the Relay service is used.

Create an account at SendGrid. Sign in and click on "Email API" in the menu and
subsequently click on the "Integration Guide" link. Then, choose "SMTP Relay",
create an API key and copy the resulting settings (Server, Ports, Username and
Password) into the `asreview_config.toml` file. It is important to continue checking
the "I've updated my settings" checkbox when it is visible **and** click on the
"Next: verify Integration" button before running the Docker containers.

It is important to verify the reply address of any email the application
will send. While being logged in on the SendGrid website, click on "Settings" in
the menu, then on "Sender Authentication" and follow the instructions.

Please note that sending emails via SendGrid with SSL requires port 465 to be
open for outbound connections on the server. Ensure that the firewall is configured
appropriately.

### Parameters in the .env file

The `.env` file contains parameters to deploy all containers. All variables that
end with the `_PORT` suffix refer to the Docker containers' external
network ports. The prefix of these variables explains for which container they
are used. Note that the external port of the frontend Docker container, the container
that is accessed directly by the end user, is 8080 and not 80. This can be changed to
80 to avoid port numbers in the URL of the ASReview LAB
application.

The value of the `WORKERS` parameter determines how many instances of the
ASReview application Gunicorn starts.

Variables prefixed with `POSTGRES` are intended for use with the PostgreSQL
database. The `_USER` and `_PASSWORD` variables represent the database user and
password, respectively. The `_DB` variable specifies the database name.

> ⚠️ The password in the `.env` file should be changed. When deploying
  Docker containers in a public environment, it is advisable to change the
  database user to something less predictable and to strengthen the password for
  enhanced security.

## Deploying to cloud provider

### Digital Ocean

The following section describes how to deploy the authenticated application with
email verification on [Digital Ocean](https://www.digitalocean.com/). The
deployment is done on a bare Droplet running Ubuntu 22.04 with 1 CPU, 2 GB of
memory and a 50 GB SSD disk. Root access is assumed.

The first consideration is a (sub)domain name. If one is available, it should
point to the IP address of the Droplet. In this description, the IP
address is used to reach the application through a browser.

Email verification (also handy for forgotten passwords) is used. For that, a
SendGrid password is required that comes from the [SendGrid setup](#sendgrid).

Connect to the Droplet over SSH, update the list of packages and install the
software for Docker:

```
$ sudo apt-get update
$ for pkg in docker.io docker-doc docker-compose docker-compose-v2 \
    podman-docker containerd runc; do sudo apt-get remove $pkg; done
$ sudo apt-get install ca-certificates curl gnupg
$ sudo install -m 0755 -d /etc/apt/keyrings
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
$ sudo chmod a+r /etc/apt/keyrings/docker.gpg
$ echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
$ sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Enter 'Y' if prompted. If questions about restarting services appear, just
select OK. To verify that everything is running, create a test Docker container:

```
$ sudo docker run hello-world
```

If all is well, a message 'Hello from Docker' should appear with other
information. Next, verify that Docker Compose is installed:

```
$ docker compose version
```

If this fails, install Docker Compose with:

```
$ sudo apt-get install docker-compose-plugin
```

The next step concerns ports. As best practice, the firewall is enabled and only
the required ports are opened. In this manual the frontend runs on port 8080 (deliberately
deviating from the usual port 80), and the backend on 8081.

```
$ sudo ufw default allow outgoing
$ sudo ufw default deny incoming
$ sudo ufw allow ssh
$ sudo ufw allow 8080
$ sudo ufw allow 8081
$ sudo ufw enable
```

Clone the ASReview Server Stack repository onto the server:

```
$ git clone https://github.com/asreview/asreview-server-stack.git
```

The next step is configuring the `.env` file in the server stack folder. Substitute
`localhost` for the Droplet's IP address and enter the reserved external port
numbers:

```
FRONTEND_DOMAIN=http://<IP address of Droplet>

FRONTEND_EXTERNAL_PORT=8080
BACKEND_EXTERNAL_PORT=8081

POSTGRES_PASSWORD="<Postgres Password>"
POSTGRES_USER="postgres"
POSTGRES_DB="asreview_db"
```

After this, the containers can be built. With the server stack root folder as the
working directory, the following command is used:

```
$ docker compose up
```

This typically takes a couple of minutes. Wait until all Docker containers are
accounted for; the backend container takes a little longer to start than the
others. The application can then be accessed by browsing to the IP address. Note
that the instance runs on HTTP, **not** HTTPS. Once this works, stop the
containers and start them again in detached mode:

```
docker compose up -d
```

### Troubleshooting

If the containers are built and running, but the application is unresponsive,
consider the following guidelines:

If there is no response whatsoever (not even a white page with a spinner), check the URL
and the protocol used in the browser. Is HTTP used instead of
HTTPS, and is the correct port being used (`http://<IP-address>:8080`)? If
so, verify that the designated ports are actually open on the server.

In the `asreview_config.toml` the `SESSION_COOKIE_SAMESITE` parameter is set to the recommended
"Lax" value. In this Docker setup, it is assumed that both the backend and
frontend can be accessed using the same domain name or IP address. When this is
not the case, the value of `SESSION_COOKIE_SAMESITE` must be set to the
string "None". Although this is an unusual setup it may help when only the
backend and database containers are deployed and a different frontend runs on
another server.

Finally, this setup does not support encryption. Its purpose is to deploy the
application as easily and quickly as possible. Dealing with certificates makes
things more complex (see the following section). Since the unencrypted HTTP protocol is used, the
`SESSION_COOKIE_SECURE` and `REMEMBER_COOKIE_SECURE` parameters in
`asreview_config.toml` are set to `false`. When the setup is adjusted to work with
certificates, it is best practice to set these values to `true`.

## Upgrading security: migrate to HTTPS

This section assumes that the steps in the preceding sections have been completed and that the application runs correctly over plain HTTP.

For any public deployment, serving the application over HTTPS is strongly recommended. In this setup, the NGINX Docker container terminates HTTPS on port 443 and redirects plain HTTP (port 80) to it. The same certificates are also passed to the ASReview Docker container, so that Gunicorn serves the backend over TLS as well.

The following extra steps are required:
* Open ports 80 and 443 on the server.
* Obtain a domain name and TLS certificates for it.
* Adjust the configuration.

### Ports

The earlier sections used port 8080 for demonstration purposes. A secured application uses the default web ports 80 and 443, which must both be open. The backend port (8081) does not need to be opened, because the backend is reachable only from within the Docker network.

The firewall tool differs per distribution. On Ubuntu and Debian, using `ufw`:
```
$ sudo ufw allow 80
$ sudo ufw allow 443
```
As in the [Quick start](#quick-start-http-without-a-domain-name), cloud providers such as Hetzner and DigitalOcean require these ports to be opened in their separate web-console firewall as well.

### Domain name and certificates

The insecure version of the ASReview web application functions without a domain name; an IP address suffices. However, for a secure version, having a domain name is highly advisable. Ensure that there is an A record in the DNS linking the domain name to the server's IP address.

A detailed explanation of domain certificates and how to obtain them falls beyond the scope of this document. Usually an IT department provides them. An alternative is to obtain free certificates from Let's Encrypt using Certbot, as briefly explained below.

On the server, install [Certbot](https://certbot.eff.org/). The installation details differ per operating system; the Certbot website provides system-specific installation instructions. Once installed, shut down any web server running on the server and issue the following command:
```
$ sudo certbot certonly --standalone
```
Provide the requested email address, agree to the terms of service and enter the domain name when prompted. After completion, Certbot produces two important files: `fullchain.pem` and `privkey.pem`. On a Linux server these files can typically be found under `/etc/letsencrypt/live/<DOMAIN_NAME>/`. Copy both files into the `asreview-server-stack` folder.

### Update the configuration

Two files need to be adjusted: `asreview_config.toml` and `docker-compose.yml`. No change to the `.env` file is required; the application derives its public address from the incoming request, so no domain name has to be configured there.

#### 1. asreview_config.toml

Because the application is now served over HTTPS, the secure-cookie parameters must be enabled. Set both to `true`:
```
SESSION_COOKIE_SECURE = true
REMEMBER_COOKIE_SECURE = true
```

#### 2. docker-compose.yml

Two services change. The `asreview` service receives the certificate files and starts Gunicorn with them. The `server` (NGINX) service receives the certificates, listens on ports 80 and 443, and uses the HTTPS NGINX configuration file `asreview_https.conf` (already included in the repository) instead of `asreview.conf`.

Replace the `asreview` service with:
```
  asreview:
    build: .
    restart: always
    depends_on:
      database:
        condition: service_healthy
    volumes:
      - project-folder:/project_folder
      - ./asreview_config.toml:/app/asreview_config.toml
      - ./fullchain.pem:/app/fullchain.pem
      - ./privkey.pem:/app/privkey.pem
    environment:
      - ASREVIEW_LAB_API_URL=/
      - ASREVIEW_LAB_CONFIG_PATH=/app/asreview_config.toml
      - ASREVIEW_LAB_SQLALCHEMY_DATABASE_URI=postgresql+psycopg2://${POSTGRES_USER}:${POSTGRES_PASSWORD}@database:5432/${POSTGRES_DB}
    ports:
      - 127.0.0.1:${BACKEND_EXTERNAL_PORT}:5006
    entrypoint: []
    command: >
      bash -c "asreview auth-tool create-db
      && (asreview task-manager
      & gunicorn --certfile /app/fullchain.pem --keyfile /app/privkey.pem -w ${WORKERS} -b \"0.0.0.0:5006\" \"asreview.webapp.app:create_app()\")"
```

Replace the `server` service with:
```
  server:
    image: nginx:1.27
    restart: always
    ports:
      - 80:80
      - 443:443
    volumes:
      - ./asreview_https.conf:/etc/nginx/nginx.conf
      - ./fullchain.pem:/etc/pemfiles/fullchain.pem
      - ./privkey.pem:/etc/pemfiles/privkey.pem
    depends_on:
      - asreview
```
Unencrypted traffic to port 80 is automatically redirected to HTTPS on port 443.

### Restart

Recreate the containers so the new configuration takes effect:
```
$ docker compose down
$ docker compose up -d
```
The application is now served over HTTPS and can be reached at `https://<domain name>`.
