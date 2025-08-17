# Deploying WordPress with MySQL Using Portainer

## Introduction

This guide provides step-by-step instructions for deploying a WordPress instance backed by a MySQL database using Portainer, a web-based management interface for Docker. The deployment uses a Docker Compose stack configuration to create two containers: one for the MySQL database (`db`) and one for WordPress (`wordpress`). This setup ensures a self-contained, scalable WordPress environment.

The configuration supports environment variable overrides for passwords and uses persistent volumes for data storage. After deployment, you'll be able to access the WordPress setup wizard to complete the installation.

**Key Features:**
- WordPress version: Latest (pulled from Docker Hub).
- MySQL version: 8.0.
- Persistent storage for database and WordPress files.
- Health checks for database readiness.
- Exposed on port 8080 by default (mapped to WordPress's internal port 80).

## Prerequisites

Before proceeding, ensure the following:
- Docker is installed and running on your server.
- Portainer is installed and accessible (e.g., via `http://<your-server-ip>:9443`). If not, install Portainer.  
- You have administrative access to Portainer.
- Basic knowledge of Docker Compose syntax.
- (Optional) Environment variables for sensitive data: Prepare values for `MYSQL_ROOT_PASSWORD` and `MYSQL_PASSWORD` if you don't want to use defaults.
- Sufficient disk space for volumes (at least 1GB recommended for initial setup).
- If exposing publicly, you'll need a Cloudflare account with Zero Trust Tunnel (cloudflared) set up.

**Security Note:** The default passwords in the script (`rootpass123` and `wppass123`) are for demonstration only. Always override them with strong, unique passwords in production.

## Deployment Steps

1. **Log in to Portainer:**  
   Access your Portainer dashboard (e.g., `http://<your-server-ip>:9443`) and select the appropriate Docker environment (local or remote).

2. **Create a New Stack:**  
   - Navigate to **Stacks** in the left sidebar.
   - Click **Add stack**.
   - Enter a name for your stack (e.g., `wordpress-stack`).  
   - Select **Web editor** as the build method.
3. **Paste the Docker Compose Script:**  
   Copy and paste the following YAML configuration into the web editor:  

   ```yaml
   version: '3.8'

   services:
     db:
       image: mysql:8.0
       container_name: wordpress-db
       restart: always
       environment:
         MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD:-rootpass123}
         MYSQL_DATABASE: wordpress
         MYSQL_USER: wpuser
         MYSQL_PASSWORD: ${MYSQL_PASSWORD:-wppass123}
       command: --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
       volumes:
         - wordpress-db-data:/var/lib/mysql
       healthcheck:
         test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
         interval: 10s
         timeout: 5s
         retries: 5
       networks:
         - wp-network

     wordpress:
       image: wordpress:latest
       container_name: wordpress
       restart: always
       environment:
         WORDPRESS_DB_HOST: db:3306
         WORDPRESS_DB_USER: wpuser
         WORDPRESS_DB_PASSWORD: ${MYSQL_PASSWORD:-wppass123}
         WORDPRESS_DB_NAME: wordpress
       ports:
         - "8080:80"
       volumes:
         - wordpress-html:/var/www/html
       depends_on:
         db:
           condition: service_healthy
       networks:
         - wp-network

   volumes:
     wordpress-db-data:
     wordpress-html:

   networks:
     wp-network:
       driver: bridge
   ```

4. **Configure Environment Variables (Optional but Recommended):**  
   - Scroll down to the **Environment variables** section in Portainer.  
   - Add variables for `MYSQL_ROOT_PASSWORD` and `MYSQL_PASSWORD` with secure values. These will override the defaults in the script.  
     Example:  
     - Name: `MYSQL_ROOT_PASSWORD`, Value: `your-strong-root-password`  
     - Name: `MYSQL_PASSWORD`, Value: `your-strong-wp-password`

5. **Deploy the Stack:**  
   - Click **Deploy the stack**.  
   - Portainer will pull the images (mysql:8.0 and wordpress:latest) if not already present and start the containers.  
   - Monitor the deployment in the **Containers** section. The `wordpress` container will wait for the `db` container's health check to pass before starting.

6. **Verify Deployment:**  
   - In Portainer, go to **Containers** and check that both `wordpress-db` and `wordpress` are running.  
   - View logs for any errors (e.g., click on the container and select **Logs**).  
   - If issues arise, ensure port 8080 is not in use by another service.

## Accessing WordPress

- Once deployed, access the WordPress setup wizard at `http://<your-server-ip>:8080` (replace `<your-server-ip>` with your server's IP or hostname). Note: Use HTTP, not HTTPS, unless you've configured SSL separately.  
- Follow the on-screen instructions to complete the WordPress installation (e.g., site title, admin user). The database details are pre-configured via environment variables.  
- After setup, the WordPress dashboard will be available at the same URL.

**Tip:** If accessing from a remote machine, ensure your firewall allows inbound traffic on port 8080 (e.g., using `ufw allow 8080` on Ubuntu).

## Exposing to the Public Using Cloudflare Tunnel (Optional)

If you want to expose WordPress securely over the internet without opening ports on your firewall, use Cloudflare Tunnel (cloudflared). This creates a secure tunnel to your server.

1. **Set Up Cloudflare Tunnel:**  
   - Sign up for Cloudflare Zero Trust (free tier available).  
   - Install cloudflared on your server: Download from https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/tunnel-guide/local/ and authenticate with `cloudflared tunnel login`.  
   - Create a tunnel: `cloudflared tunnel create <tunnel-name>`.  
   - Configure the tunnel to point to your WordPress service: Edit `~/.cloudflared/config.yml` with:  
     ```yaml
     tunnel: <tunnel-name>
     credentials-file: /root/.cloudflared/<tunnel-uuid>.json
     ingress:
       - hostname: <your-fqdn>  # e.g., wordpress.example.com
         service: http://localhost:8080
       - service: http_status:404
         originRequest:
           noTLSVerify: true
     ```  
   - Run the tunnel: `cloudflared tunnel run <tunnel-name>`.  
   - Add a DNS record in Cloudflare for `<your-fqdn>` pointing to the tunnel.

2. **Modify wp-config.php for HTTPS:**  
   After WordPress is installed and the tunnel is active, update `wp-config.php` to use your public FQDN with HTTPS. This prevents mixed content issues and ensures proper redirects.  
   - Exec into the WordPress container via Portainer: Go to **Containers** > Select `wordpress` > **Console** > Connect (use `/bin/sh`).  
   - Run the following command inside the container (replace `<your-fqdn>` with your actual domain, e.g., `wordpress.example.com`):  
     ```bash
     sed -i "/\/\* That's all, stop editing! Happy publishing\. \*\//i define('WP_HOME', 'https://<your-fqdn>');\ndefine('WP_SITEURL', 'https://<your-fqdn>');" /var/www/html/wp-config.php
     ```  
     **Note:** Both `WP_HOME` and `WP_SITEURL` should use `https://` for consistency and to avoid redirect loops. Restart the WordPress container in Portainer if changes don't apply immediately.

3. **Access Publicly:**  
   Visit `https://<your-fqdn>` to access WordPress over the secure tunnel.

**Security Considerations:** Enable HTTPS enforcement in Cloudflare. Use strong authentication and monitor for vulnerabilities in WordPress plugins/themes.

## Additional Notes

- **Multiple WordPress Deployments:** If deploying more than one WordPress instance on the same host, modify the `container_name` fields to ensure uniqueness (e.g., `wordpress-db-1` and `wordpress-1` for the first, `wordpress-db-2` and `wordpress-2` for the second). Alternatively, remove the `container_name` lines entirely to let Docker auto-generate unique names. Also, use different port mappings (e.g., `"8081:80"`) and volume names to avoid conflicts.
- **Scaling and Customization:** For production, consider adding a reverse proxy like Nginx for SSL termination or integrating with a CDN. Update images periodically via Portainer to get the latest security patches.
- **Backup and Restore:** Regularly back up volumes (`wordpress-db-data` and `wordpress-html`) using Portainer's volume management or tools like `docker volume ls` and `rsync`.
- **Troubleshooting Common Issues:**  
  - Database connection errors: Check logs for password mismatches or health check failures.  
  - Permission issues: Ensure the WordPress volume is writable (chown if needed inside the container).  
  - If the stack fails to deploy, validate YAML syntax or increase resource limits in Docker settings.  
- **Environment Variables:** Defaults are provided, but always override passwords. You can also add more WordPress env vars (e.g., `WORDPRESS_DEBUG: true`) in the stack configuration for debugging.

This setup provides a robust, containerized WordPress environment. For advanced configurations, refer to the official WordPress Docker documentation or Portainer guides. If you encounter issues, provide container logs for further assistance.

---

# Resolving WordPress Database Connection Issues with Docker MySQL Container

## Overview

This guide explains how to resolve WordPress errors when it cannot establish a database connection while running on Docker. It assumes you already have a MySQL container (named `florentine-db`) running and WordPress connected to it.

## Prerequisites

* Docker installed and running
* `florentine-db` MySQL container up and running
* WordPress container configured to connect to `florentine-db`
* MySQL root password available
* Knowledge of MySQL user credentials for WordPress (`wpuser`)

## Steps to Fix Database Connection

### 1. Access the MySQL Container

Run the following command to open a MySQL prompt inside the running container:

```bash
docker exec -it florentine-db mysql -u root -p
```

You will be prompted to enter the MySQL root password:

```
Enter password: <mysql-password>
```

### 2. Update the WordPress Database User

Once inside the MySQL prompt, execute the following commands to ensure the WordPress user (`wpuser`) can connect from any host using the native password plugin:

```sql
ALTER USER 'wpuser'@'%' IDENTIFIED WITH mysql_native_password BY '<mysql-password>' REQUIRE NONE;
FLUSH PRIVILEGES;
```

**Explanation:**

* `ALTER USER`: Updates the authentication method and password for `wpuser`.
* `mysql_native_password`: Ensures compatibility with WordPress.
* `FLUSH PRIVILEGES`: Reloads MySQL privileges to apply changes immediately.

### 3. Exit MySQL Prompt

Exit the MySQL prompt:

```sql
EXIT;
```

### 4. Restart Docker Containers

To apply the changes, restart both your MySQL and WordPress containers:

```bash
docker restart florentine-db
docker restart <wordpress-container-name>
```

**Note:** Replace `<wordpress-container-name>` with the actual name of your WordPress container.

## Validation

After restarting, verify that WordPress can connect to the database:

1. Open your WordPress site in a browser.
2. If the site loads correctly without the `Error establishing a database connection` message, the issue is resolved.

