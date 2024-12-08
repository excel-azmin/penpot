```
networks:
  penpot:
  traefik-public:
    external: true

volumes:
  penpot_postgres_v15:
  penpot_assets:

services:
  penpot-frontend:
    image: "penpotapp/frontend:2.3.3"
    restart: always
    ports:
      - 9001:80
    volumes:
      - penpot_assets:/opt/data/assets
    depends_on:
      - penpot-backend
      - penpot-exporter
    networks:
      - penpot
      - traefik-public
    deploy:
      replicas: 1
      restart_policy:
        condition: on-failure
        delay: 5s
      labels:
        traefik.docker.network: traefik-public
        traefik.enable: "true"
        traefik.constraint-label: traefik-public
        traefik.http.routers.draw-http.rule: Host(${SITES:?No sites set})
        traefik.http.routers.draw-http.entrypoints: http
        traefik.http.routers.draw-http.middlewares: https-redirect
        traefik.http.routers.draw-https.rule: Host(${SITES})
        traefik.http.routers.draw-https.entrypoints: https
        traefik.http.routers.draw-https.tls: "true"
        traefik.http.routers.draw-https.tls.certresolver: le
        traefik.http.services.draw.loadbalancer.server.port: "80"
    environment:
      ## Relevant flags for frontend:
      ## - demo-users
      ## - login-with-github
      ## - login-with-gitlab
      ## - login-with-google
      ## - login-with-ldap
      ## - login-with-oidc
      ## - login-with-password
      ## - registration
      ## - webhooks
      ##
      ## You can read more about all available flags on:
      ## https://help.penpot.app/technical-guide/configuration/#advanced-configuration

      - PENPOT_FLAGS=enable-registration enable-login-with-password

  penpot-backend:
    image: "penpotapp/backend:2.3.3"
    restart: always
    volumes:
      - penpot_assets:/opt/data/assets
    depends_on:
      - penpot-postgres
      - penpot-redis
    networks:
      - penpot
    environment:
      ## Relevant flags for backend:
      ## - demo-users
      ## - email-verification
      ## - log-emails
      ## - log-invitation-tokens
      ## - login-with-github
      ## - login-with-gitlab
      ## - login-with-google
      ## - login-with-ldap
      ## - login-with-oidc
      ## - login-with-password
      ## - registration
      ## - secure-session-cookies
      ## - smtp
      ## - smtp-debug
      ## - telemetry
      ## - webhooks
      ## - prepl-server
      ##
      ## You can read more about all available flags and other
      ## environment variables for the backend here:
      ## https://help.penpot.app/technical-guide/configuration/#advanced-configuration

      - PENPOT_FLAGS=enable-registration enable-login-with-password disable-email-verification enable-smtp enable-prepl-server

      ## Penpot SECRET KEY. It serves as a master key from which other keys for subsystems
      ## (eg http sessions, or invitations) are derived.
      ##
      ## If you leave it commented, all created sessions and invitations will
      ## become invalid on container restart.
      ##
      ## If you going to uncomment this, we recommend to use a trully randomly generated
      ## 512 bits base64 encoded string here.  You can generate one with:
      ##
      ## python3 -c "import secrets; print(secrets.token_urlsafe(64))"

      # - PENPOT_SECRET_KEY=my-insecure-key

      ## The PREPL host. Mainly used for external programatic access to penpot backend
      ## (example: admin). By default it will listen on `localhost` but if you are going to use
      ## the `admin`, you will need to uncomment this and set the host to `0.0.0.0`.

      # - PENPOT_PREPL_HOST=0.0.0.0

      ## Public URI. If you are going to expose this instance to the internet and use it
      ## under a different domain than 'localhost', you will need to adjust it to the final
      ## domain.
      ##
      ## Consider using traefik and set the 'disable-secure-session-cookies' if you are
      ## not going to serve penpot under HTTPS.

      - PENPOT_PUBLIC_URI=https://draw.arcapps.org

      ## Database connection parameters. Don't touch them unless you are using custom
      ## postgresql connection parameters.

      - PENPOT_DATABASE_URI=postgresql://penpot-postgres/penpot
      - PENPOT_DATABASE_USERNAME=penpot
      - PENPOT_DATABASE_PASSWORD=penpot

      ## Redis is used for the websockets notifications. Don't touch unless the redis
      ## container has different parameters or different name.

      - PENPOT_REDIS_URI=redis://penpot-redis/0

      ## Default configuration for assets storage: using filesystem based with all files
      ## stored in a docker volume.

      - PENPOT_ASSETS_STORAGE_BACKEND=assets-fs
      - PENPOT_STORAGE_ASSETS_FS_DIRECTORY=/opt/data/assets

      ## Also can be configured to to use a S3 compatible storage
      ## service like MiniIO. Look below for minio service setup.

      # - AWS_ACCESS_KEY_ID=<KEY_ID>
      # - AWS_SECRET_ACCESS_KEY=<ACCESS_KEY>
      # - PENPOT_ASSETS_STORAGE_BACKEND=assets-s3
      # - PENPOT_STORAGE_ASSETS_S3_ENDPOINT=http://penpot-minio:9000
      # - PENPOT_STORAGE_ASSETS_S3_BUCKET=<BUKET_NAME>

      ## Telemetry. When enabled, a periodical process will send anonymous data about this
      ## instance. Telemetry data will enable us to learn how the application is used,
      ## based on real scenarios. If you want to help us, please leave it enabled. You can
      ## audit what data we send with the code available on github.

      - PENPOT_TELEMETRY_ENABLED=true

      ## Example SMTP/Email configuration. By default, emails are sent to the mailcatch
      ## service, but for production usage it is recommended to setup a real SMTP
      ## provider. Emails are used to confirm user registrations & invitations. Look below
      ## how the mailcatch service is configured.

      - PENPOT_SMTP_DEFAULT_FROM=${PENPOT_SMTP_USERNAME}
      - PENPOT_SMTP_DEFAULT_REPLY_TO=${PENPOT_SMTP_USERNAME}
      - PENPOT_SMTP_HOST=${PENPOT_SMTP_HOST}
      - PENPOT_SMTP_PORT=465
      - PENPOT_SMTP_USERNAME=${PENPOT_SMTP_USERNAME}
      - PENPOT_SMTP_PASSWORD=${PENPOT_SMTP_PASSWORD}
      - PENPOT_SMTP_TLS=true
      - PENPOT_SMTP_SSL=true

  penpot-exporter:
    image: "penpotapp/exporter:2.3.3"
    restart: always
    networks:
      - penpot
    environment:
      # Don't touch it; this uses an internal docker network to
      # communicate with the frontend.
      - PENPOT_PUBLIC_URI=http://penpot-frontend
      ## Redis is used for the websockets notifications.
      - PENPOT_REDIS_URI=redis://penpot-redis/0

  penpot-postgres:
    image: "postgres:15"
    restart: always
    stop_signal: SIGINT
    volumes:
      - penpot_postgres_v15:/var/lib/postgresql/data
    networks:
      - penpot
    environment:
      - POSTGRES_INITDB_ARGS=--data-checksums
      - POSTGRES_DB=penpot
      - POSTGRES_USER=penpot
      - POSTGRES_PASSWORD=penpot
  penpot-redis:
    image: redis:7
    restart: always
    networks:
      - penpot
  penpot-mailcatch:
    image: sj26/mailcatcher:latest
    restart: always
    expose:
      - '1025'
    ports:
      - "1080:1080"
    networks:
      - penpot

```
