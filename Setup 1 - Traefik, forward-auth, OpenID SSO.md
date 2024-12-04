To set up a **Traefik** reverse proxy in a Dockerized environment with **Let's Encrypt SSL** certificates for **domain.tld** and all of its subdomains, and to redirect traffic to a standard error page for the base domain, you will need to configure several key components:

1. **Docker Compose** for setting up Traefik and other services.
2. **Traefik Configuration File** to enable **Let's Encrypt**, handle routing, and configure the SSL.
3. Configuration to forward traffic to the **Synology SSO server**.
4. Ensure Traefik handles subdomains and redirects traffic from `domain.tld` to an error page.

# Assumptions:
- The domain is dynamic, so you will need a **dynamic DNS (DDNS)** setup to keep the domain pointing to your server.
- The router is already forwarding ports `80` and `443` to the internal docker server's ports `11420` and `11422`.
- Your Synology SSO server is accessible via `subdomain.domain.tld` (for example `sso.domain.tld`).
- Traefik will be running in a Docker container and will route traffic to the SSO server and other subdomains.

# Sources

[Here's an excellent source](https://www.benjaminrancourt.ca/a-complete-traefik-configuration/) for a setup which I took some information from for the following description.

# 1. **Dynamic DNS Setup**

Since your domain `domain.tld` has a dynamic IP, use a Dynamic DNS (DDNS) provider that automatically updates the DNS records whenever your IP changes. Many domain registrars offer DDNS services or you can use third-party services like **Cloudflare**, **DuckDNS**, etc.

For **Let's Encrypt** to function properly, your DNS provider must allow Traefik to verify domain ownership via HTTP-01 or DNS-01 challenge. Ensure your DDNS provider supports this.

# 2. **Docker Compose File (`docker-compose.yml`)**

The `docker-compose.yml` will define the Traefik container along with the other services, including your **Synology SSO server** and an error page.

```yaml
services:
  # Traefik service (reverse proxy with Let's Encrypt)
  traefik:
    image: traefik:stable
    container_name: web-tools-traefik
    command: # this shouldn't be used if static conf file is used (see volumes section); see also https://doc.traefik.io/traefik/getting-started/configuration-overview/#the-static-configuration
      #- "--api.insecure=true"  # Enable the Traefik dashboard (optional, only for testing)
      #- "--providers.docker=true"  # Enable Docker provider
      #- "--entryPoints.web.address=:80"  # HTTP entry point for Let's Encrypt HTTP challenge
      #- "--entryPoints.websecure.address=:443"  # HTTPS entry point
      #- "--certificateResolvers.letsencrypt.acme.httpChallenge.entryPoint=web"  # HTTP-01 challenge for Let's Encrypt
      #- "--certificateResolvers.letsencrypt.acme.email=your-email@domain.tld"  # Email for Let's Encrypt registration
      #- "--certificateResolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"  # Persistent storage for certificates
      #- "--log.level=INFO"  # Log level (adjust for production)
      #- "--accesslog=true"  # Enable access logs
    ports:
      - "11420:80"  # Expose HTTP port
      - "11422:443"  # Expose HTTPS port
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro  # To allow Traefik to talk to Docker API; see https://doc.traefik.io/traefik/providers/docker/#provider-configuration
      - ./letsencrypt:/letsencrypt  # Store Let's Encrypt certificates  # (mkdir -p ./letsencrypt && touch $_/acme.json && chmod 600 $_)
      - ./config/traefik:/etc/traefik # For static configuration; see also https://doc.traefik.io/traefik/getting-started/configuration-overview/#configuration-file
    networks:
      - host

  # Synology SSO Server (example)
  # sso-server:
  #   image: your-sso-image  # Replace with your actual Synology SSO image
  #   container_name: sso-server
  #   environment:
  #     - "SSO_SERVER=true"
  #   labels:
  #     - "traefik.enable=true"
  #     - "traefik.http.routers.sso.rule=Host(`sso.domain.tld`)"
  #     - "traefik.http.routers.sso.entrypoints=websecure"
  #     - "traefik.http.services.sso.loadbalancer.server.port=11420"  # Internal port of your SSO server
  #   networks:
  #     - web

  # Error page container
  # error-page:
  #   image: nginx:alpine  # You can use any image that serves an error page
  #   container_name: error-page
  #   volumes:
  #     - "./error-page:/usr/share/nginx/html"
  #   labels:
  #     - "traefik.enable=true"
  #     - "traefik.http.routers.error.rule=Host(`domain.tld`)"  # Redirect base domain to error page
  #     - "traefik.http.routers.error.entrypoints=websecure"
  #     - "traefik.http.services.error.loadbalancer.server.port=80"
  #   networks:
  #     - web

# networks:
#   web:
#     external: true  # Assuming you have an external network for your services
```

## Key Points:
- **Traefik Service**: It listens on ports `11420` and `11422`, which are mapped to HTTP and HTTPS, respectively. Traefik handles automatic SSL renewal using Let's Encrypt.
  - The `acme.json` file will store the certificates generated by Let's Encrypt. Ensure that this file is writable.
  - The `certificatesResolvers.le.acme.httpChallenge.entryPoint=web` defines that the HTTP-01 challenge is handled via port `80`.
  
- **Synology SSO Service**: This service is your Synology SSO server that will be exposed to `sso.domain.tld`. Replace the image with the correct one for your Synology SSO server.

- **Error Page Service**: This uses a basic `nginx` container that serves a static error page for `domain.tld`. You can replace the HTML content of the error page as needed.

# 3. **Traefik Configuration File (traefik.yml)**

This file is optional and for [static configuration](https://doc.traefik.io/traefik/getting-started/configuration-overview/#the-static-configuration), but it allows for more advanced configuration and settings outside of the `docker-compose.yml`. Since Traefik configuration is already primarily handled by labels in the compose file, this is for additional custom configurations.

```yaml
api:  # see https://doc.traefik.io/traefik/operations/api/
  insecure: true  # Exposes the dashboard on port 8080 (optional, for testing purposes)
  dashboard: true

# log:
#   level: INFO

accessLog: {}

providers:
  docker:  # see https://doc.traefik.io/traefik/providers/docker/#provider-configuration
    endpoint: "unix:///var/run/docker.sock"
    exposedbydefault: false
  file:
    filename: /etc/traefik/config.yml

entryPoints:  # Used by certificatesResolvers and for specific definition (otherwise implicit by standard). Must be defined for custom ports anyway, and enchances clarity even if not required.
  web:
    address: ":80"
  websecure:
    address: ":443"

certificateResolvers:
  letsencrypt:
    acme:
      email: your-email@example.com  # Replace with your e-mail address
      storage: /letsencrypt/acme.json  # (mkdir -p ./letsencrypt && touch $_/acme.json && chmod 600 $_)
      httpChallenge:
        entryPoint: web  # Challenge through HTTP (port 80)
```

This configuration file is used for storing **ACME (Let's Encrypt)** configuration and additional logging. The storage path for `acme.json` should be persistent, hence it’s mapped as a volume.

# 4. **Traefik Provider File (config.yml)**
This file is optional and for [dynamic configuration](https://doc.traefik.io/traefik/getting-started/configuration-overview/#the-dynamic-configuration) and allows for more advanced configuration and settings outside of the `docker-compose.yml`. Since Traefik configuration is already primarily handled by labels in the compose file, this is for, e.g., services/providers outside of docker, like, e.g., a standalone SSO server.

```yaml
http:
  routers:
    SSO:
      rule: "Host(`sso.domain.tld`)"
      entrypoints: websecure
  services:
    SSO:
      loadbalancer:
        server:
          port: 11420
  middlewares:
    # Always redirect to https
    redirect-to-https:
      redirectScheme:
        scheme: https
    # Secure headers
    HSTS:
      headers:
        frameDeny: true
        browserXssFilter: true
        # Add the following to docker-compose to use as alternative:
        # labels:
        #  - "traefik.http.middlewares.HSTS.headers.framedeny=true"
        #  - "traefik.http.middlewares.HSTS.headers.browserxssfilter=true"
```

# 5. **Error Page (HTML Content)**

You can place a simple `index.html` for the error page that will be served by `nginx`:

```html
<!-- error-page/index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>404 Error</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      display: flex;
      justify-content: center;
      align-items: center;
      height: 100vh;
      margin: 0;
      background-color: #f0f0f0;
      color: #333;
    }
    h1 {
      font-size: 3em;
    }
  </style>
</head>
<body>
  <h1>404 - Page Not Found</h1>
</body>
</html>
```

This simple error page will be served when accessing `domain.tld`.

# 6. **Important Considerations**

- **DNS**: Ensure that your DNS provider properly supports wildcard subdomains (`*.domain.tld`) and points to the external IP of your router, which forwards ports `80` and `443` to the Dockerized Traefik.
- **Dynamic DNS**: Ensure that your dynamic DNS service is properly updating the IP of your domain as your external IP changes.

# 7. **Final Steps**

1. **Deploy the Docker Containers**:
   - Run the `docker-compose.yml` file by navigating to the directory containing it and running:
     ```bash
     docker-compose up -d
     ```

2. **Verify Let's Encrypt SSL**:
   - Traefik will automatically request certificates from Let's Encrypt for `domain.tld` and its subdomains, such as `sso.domain.tld`. You can check the logs to see the certificate request process:
     ```bash
     docker logs -f traefik
     ```

3. **Verify Error Page**:
   - Access `https://domain.tld` and you should be redirected to the error page.

4. **Check for Valid SSL**:
   - Access `https://sso.domain.tld` to ensure that SSL is correctly configured, and the SSO server is accessible.

This setup ensures that **Traefik** handles automatic SSL certificate management via **Let's Encrypt** for all subdomains and `domain.tld`, routes traffic to your Synology SSO server, and serves a custom error page for the base domain (`domain.tld`).
