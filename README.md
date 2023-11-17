# How to install Outline >0.72.0 (with Traefik)

## Introduction

This guide allows you to install [Outline](https://www.getoutline.com/) (Notion self hosted alternative) without the fuss. Don't be in the hurry, <mark style="color:purple;">**follow steps and instructions calmly**</mark> (said Dumbledore) to avoid giving up or restart from scratch.

{% hint style="info" %}
Versions above 0.72.0 now offer the possibility to not have an S3/Minio instance, and host images in local.
{% endhint %}

Estimated time: around 30 minutes

***

## Prerequisites

* [Docker](https://docs.docker.com/get-docker/),
* Reverse proxy (thus, a domain name)
* [Redis](https://redis.io/) (selfhosted alternative to Amazon DynamoDB),
* [PostgreSQL](https://www.postgresql.org/) database,
* An OpenID Connect (OIDC) provider ([Keycloak](https://www.keycloak.org/), [Authelia](https://www.authelia.com/), [Authentik](https://goauthentik.io/), etc.).

“That's all.”

{% hint style="danger" %}
~~Don't skip anything! You will be stuck at some point!~~

~~I did, and lost my time 🥲~~

I wrote that when Amazon S3/Minio was a requirement, now, it should be more seamless.
{% endhint %}

## How is this tutorial structured?

I give the complete Docker Compose in the part called [#full-docker-compose](./#full-docker-compose "mention"). Before this part, I give you part of this Docker Compose, and give explanation (almost for beginners, if you're more experienced, you can skip these sections).

## Optional - Docker network

I like to have linked containers under the same Docker network. I'm creating the `outline-network`. Feel free to not have one.

## First, database

Starting with the simplest. PostgreSQL is the recommended DBMS, you may try with MariaDB, but I never tried. Outline suggests at least, the version 12.0. I use the version 16.1

<pre class="language-yaml"><code class="lang-yaml">version: "3.9"

services:
<strong>  db:
</strong>    image: postgres:16.1
    container_name: outline-db
    environment:
      POSTGRES_USER: "outline"
      POSTGRES_PASSWORD: "outline" # Feel free to use secrets or a separate .env
      POSTGRES_DB: "outline"
    volumes:
      - &#x3C;path on your server>:/var/lib/postgresql/data/
    # v--- Optional from here ---v #
    networks:
      - outline-network
 
networks:
  outline-network:
    external: true
</code></pre>

### Explanations

We ask Docker to pull a PostgreSQL image, create a container with it and name it “outline-db”. A database named `outline` with a user `outline` will be created automatically with the password `outline`. We mount a physical location to the container location to avoid loses when restarting the container.

## Second, Redis

Outline asks as a minimum version, the 4.0. We will take the 7. It works.

```yaml
version: "3.9"

services:
  redis:
    image: redis
    container_name: outline-redis
    hostname: outline-redis
    volumes:
      - "<path on your server>/redis.conf:/redis.conf:rw"
    command: ["redis-server", "/redis.conf"]
    # v--- Optional from here ---v #
    networks:
      - outline-network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 30s
      retries: 3
     
networks:
  outline-network:
    external: true
```

### Explanations

Redis offers a high efficiency storage. Here, we ask Docker to pull a Redis image, create a container with it and name it “outline-redis”. We mount a physical location to the container location to avoid loses when restarting the container, here, only one file is necessary `redis.conf`. Optionally, you can add a user and a password; but as it is not exposed, I won't add anything.

## Preparing the `.env` file

[Get the .env file](https://github.com/outline/outline/blob/main/.env.sample) that we will use as template. Right, let's go step by step to fill this bloody .env file. I removed included comments to have a clear view.

### Step 1: random keys

First, on lines 7 and 11, we need to generate **two random strings** and put them as value of `SECRET_KEY` and `UTILS_SECRET`. As indicated, you may use the command `openssl rand -hex 32`.

#### After doing so, here's an example of the expected result:

```
SECRET_KEY=66bd96bbec546f8729d6df027c93e97094c05c7800d17c04b88820aec1d5cc3c
UTILS_SECRET=9470f2d8e121f87019587cc46dfc3a39990e6f7c34b7d681cb8fc48e1832a132
```

### Step 2: database connection

We obviously need to connect our Outline instance to our PostgreSQL database (lines 15-20). Here, we'll replace `localhost` by `outline-db` (or the name of your PostgreSQL container). Here's an example of the expected result:

```yaml
DATABASE_URL=postgres://outline:outline@outline-db:5432/outline
DATABASE_CONNECTION_POOL_MIN= # Yes, you may it them empty
DATABASE_CONNECTION_POOL_MAX= # Yes, you may it them empty
PGSSLMODE=disable
```

* I uncomment the line 20, `PGSSLMODE=disable` as Outline and PostgreSQL communicate without SSL inside the Docker network, which is not a serious security issue.
* You may comment the line 16, `DATABASE_URL_TEST` as we won't use a test database.

### Step 3: Redis connection

Same thing, but for Redis (line 23).

```
REDIS_URL=redis://outline-redis:6379
```

Just replace `localhost` by the Docker Redis container's name.

### **Step 4: setting up Outline URL**

Lines 33-34 ask for an URL, the Outline URL, what you will exactly type in your browser bar.

```
URL=https://outline.yoursuperdomain.tld
PORT=3000
```

Traefik will set up the URL for us. Keep PORT equals 3000, it is used to indicate which port Outline should listen. Change it if you wish but change accordingly on your reverse proxy solution - for Traefik, to change the port in labels.

### Step 5: authentication provider

Now, lines 101-105 invite us to type OIDC information. I have Keycloak, but information asked are the same for all OIDC providers. Create a client as usual, mine is named ouline. The secret is generated by Keycloak (Clients details > Credentials).

<figure><img src=".gitbook/assets/Capture d’écran (8).png" alt=""><figcaption></figcaption></figure>

URLs can be found in the realm configuration. If you use Keycloak, they are the same, just replace the domain and the realm name `master` into whatever you use.

```
OIDC_CLIENT_ID=outline
OIDC_CLIENT_SECRET=mysupersecretomg
OIDC_AUTH_URI=https://sso.mysuperdomain.tld/realms/master/protocol/openid-connect/auth
OIDC_TOKEN_URI=https://sso.mysuperdomain.tld/realms/master/protocol/openid-connect/token
OIDC_USERINFO_URI=https://sso.mysuperdomain.tld/realms/master/protocol/openid-connect/userinfo
```

## Outline itself

```yaml
version: "3.9"

services:
  outline:
    image: docker.getoutline.com/outlinewiki/outline:latest
    container_name: outline
    hostname: outline
    restart: unless-stopped
    env_file: ./outline.env
    depends_on:
      - postgres
      - redis
    volumes:
      - "<path on your server>:/var/lib/outline/data:rw"
    # v--- Optional if you don't use Traefik ---v #
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.outline.entrypoints=https"
      - "traefik.http.routers.outline.rule=Host(`outline.yoursuperdomain.tld`)"
      - "traefik.http.routers.outline.tls=true"
      - "traefik.http.services.outline.loadbalancer.server.port=3000"
      - "traefik.docker.network=outline-network"
    # ^--- Optional ends ---^ #
    # v--- Optional from here ---v #
    networks:
      - outline-network

networks:
  outline-network:
    external: true
```

### Notes

Note, the `traefik.http.services.outline.loadbalancer.server.port=3000` to specify which port Traefik should contact Outline. If you changed it in the .env file, change it also here.

## Full Docker Compose

That's my organization:

```
└── Docker Compose
    ├── outline
    │   ├── outline.env
    │   └── outline.yaml
    └── traefik.yaml
```

<pre class="language-yaml"><code class="lang-yaml"><strong>version: "3.9"
</strong>
services:
  outline:
    image: docker.getoutline.com/outlinewiki/outline:latest
    container_name: outline
    hostname: outline
    restart: unless-stopped
    env_file: ./outline.env
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.outline.entrypoints=https"
      - "traefik.http.routers.outline.rule=Host(`outline.yoursuperdomain.tld`)"
      - "traefik.http.routers.outline.tls=true"
      - "traefik.http.services.outline.loadbalancer.server.port=3000"
      - "traefik.docker.network=outline-network"
    volumes:
      - "&#x3C;path on your server>:/var/lib/outline/data:rw"
    depends_on:
      - postgres
      - redis
    # v--- Optional from here ---v #
    networks:
      - outline-network
    # ^--- Optional ends here ---^ #
    
  redis:
    image: redis:latest
    container_name: outline-redis
    hostname: outline-redis
    volumes:
      - "&#x3C;path on your server>/redis.conf:/redis.conf:rw"
    command: ["redis-server", "/redis.conf"]
    # v--- Optional from here ---v #
    networks:
      - outline-network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 30s
      retries: 3
    # ^--- Optional ends here ---^ #

  postgres:
    image: postgres:latest
    container_name: outline-db
    hostname: outline-db
    environment:
      POSTGRES_USER: "outline"
      POSTGRES_PASSWORD: "outline"
      POSTGRES_DB: "outline"
    volumes:
      - "&#x3C;path on your server>:/var/lib/postgresql/data:rw"
    networks:
      - outline-network
    # v--- Optional from here ---v #
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "outline", "-d", "outline"]
      interval: 30s
      timeout: 20s
      retries: 3
    # ^--- Optional ends here ---^ #

# v--- Optional from here ---v #
networks:
  outline-network:
    external: true
# ^--- Optional ends here ---^ #
</code></pre>

## Complete .env file

This is what we changed (without comments).

```yaml
NODE_ENV=production

SECRET_KEY=66bd96bbec546f8729d6df027c93e97094c05c7800d17c04b88820aec1d5cc3c
UTILS_SECRET=9470f2d8e121f87019587cc46dfc3a39990e6f7c34b7d681cb8fc48e1832a132

DATABASE_URL=postgres://outline:outline@outline-db:5432/outline
DATABASE_CONNECTION_POOL_MIN=
DATABASE_CONNECTION_POOL_MAX=
PGSSLMODE=disable

REDIS_URL=redis://outline-redis:6379

URL=https://outline.yoursuperdomain.tld
PORT=3000

OIDC_CLIENT_ID=outline
OIDC_CLIENT_SECRET=VbUil3dbTeukSLPj2U2Kg1DCvnLc2eYP
OIDC_AUTH_URI=https://sso.yoursuperdomain.tld/realms/master/protocol/openid-connect/auth
OIDC_TOKEN_URI=https://sso.yoursuperdomain.tld/realms/master/protocol/openid-connect/token
OIDC_USERINFO_URI=https://sso.yoursuperdomain.tld/realms/master/protocol/openid-connect/userinfo
```

## The End

That's all, pal. You did it. Head to https://outline.yoursuperdomain.tld and you should see&#x20;

<figure><img src=".gitbook/assets/Capture d’écran (10).png" alt=""><figcaption><p>Login page.</p></figcaption></figure>

Sadly, you can't remove Slack (yet?). If you dare to remove it, Outline will scream.

## Possible enhancements

* Set the language in the `.env` with `DEFAULT_LANGUAGE=xx_XX`. I'm not sure where Outline uses it - because it changes the language with the language set in your browser. Just in case, change it here too.
* Use secrets or one .env file for everything (I mean, for environment variables I set up directly in the Docker Compose file).

## Conclusion

Feel free to make a pull request [here](https://github.com/DimitriDR/dead-simple-tutorials-outline), if you find a typo or a thing that deserves to be more detailed.
