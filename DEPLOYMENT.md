# VPS Deployment Guide for Forked Logto with Docker

This guide will help you deploy your customized Logto (with numeric username support) to a VPS using Docker.

## Prerequisites

- A VPS running Ubuntu 20.04+ or Debian 11+ (DigitalOcean, Linode, Vultr, Hetzner, etc.)
- Domain name pointed to your VPS IP address
- SSH access to your VPS
- At least 2GB RAM and 10GB storage

## Step 1: Prepare Your VPS

1. **SSH into your VPS:**
   ```bash
   ssh root@your-vps-ip
   ```

2. **Update system packages:**
   ```bash
   apt update && apt upgrade -y
   ```

3. **Install Docker and Docker Compose:**
   ```bash
   # Install Docker
   curl -fsSL https://get.docker.com | sh
   
   # Install Docker Compose plugin
   apt-get install docker-compose-plugin -y
   
   # Verify installation
   docker --version
   docker compose version
   ```

4. **Install Git and other utilities:**
   ```bash
   apt install git nginx certbot python3-certbot-nginx -y
   ```

## Step 2: Deploy Your Forked Logto

1. **Clone your forked repository with your custom changes:**
   ```bash
   cd /opt
   git clone https://github.com/YOUR_USERNAME/logto-koptnb.git
   cd logto-koptnb
   ```

2. **Create production environment file:**
   ```bash
   nano .env.production
   ```
   
   Add the following (customize values):
   ```env
   # Database Configuration
   POSTGRES_USER=logto
   POSTGRES_PASSWORD=your_strong_password_here
   POSTGRES_DB=logto
   DB_URL=postgres://logto:your_strong_password_here@postgres:5432/logto
   
   # Application Configuration
   PORT=3001
   ADMIN_PORT=3002
   NODE_ENV=production
   
   # Your domain configuration
   ENDPOINT=https://auth.yourdomain.com
   ADMIN_ENDPOINT=https://admin.yourdomain.com
   
   # Security
   TRUST_PROXY_HEADER=1
   
   # Optional: Session secrets (generate random strings)
   OIDC_COOKIE_KEYS=your_random_key_1,your_random_key_2
   ```

3. **Create production docker-compose file:**
   ```bash
   nano docker-compose.prod.yml
   ```
   
   Add this configuration that builds from YOUR local code:
   ```yaml
   services:
     postgres:
       image: postgres:17-alpine
       container_name: logto-db
       restart: always
       environment:
         POSTGRES_USER: ${POSTGRES_USER}
         POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
         POSTGRES_DB: ${POSTGRES_DB}
       volumes:
         - postgres_data:/var/lib/postgresql/data
       networks:
         - logto-network
       healthcheck:
         test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
         interval: 10s
         timeout: 5s
         retries: 5
   
     logto:
       build:
         context: .
         dockerfile: Dockerfile
         args:
           # Optional build args if needed
           dev_features_enabled: ${DEV_FEATURES_ENABLED:-}
       container_name: logto-app
       restart: always
       depends_on:
         postgres:
           condition: service_healthy
       ports:
         - "127.0.0.1:3001:3001"
         - "127.0.0.1:3002:3002"
       environment:
         DB_URL: ${DB_URL}
         ENDPOINT: ${ENDPOINT}
         ADMIN_ENDPOINT: ${ADMIN_ENDPOINT}
         PORT: ${PORT}
         ADMIN_PORT: ${ADMIN_PORT}
         NODE_ENV: ${NODE_ENV}
         TRUST_PROXY_HEADER: ${TRUST_PROXY_HEADER}
       volumes:
         # Mount for custom connectors if needed
         - ./custom-connectors:/etc/logto/packages/core/connectors
       networks:
         - logto-network
       command: ["sh", "-c", "npm run cli db seed -- --swe && npm start"]
   
   networks:
     logto-network:
       driver: bridge
   
   volumes:
     postgres_data:
       driver: local
   ```

## Step 3: Build and Run Your Custom Logto

1. **Load environment variables and build the Docker image from your code:**
   ```bash
   # Load environment variables
   export $(cat .env.production | xargs)
   
   # Build your custom Logto image (this uses YOUR modified code)
   docker compose -f docker-compose.prod.yml build
   
   # This build process will:
   # - Use your modified regex.ts with numeric username support
   # - Use your modified form.ts without number restriction
   # - Include all your custom changes
   ```

2. **Start the services:**
   ```bash
   docker compose -f docker-compose.prod.yml up -d
   
   # Check logs
   docker compose -f docker-compose.prod.yml logs -f
   ```

3. **Verify services are running:**
   ```bash
   docker ps
   # Should show both logto-app and logto-db containers
   ```

## Step 4: Configure Nginx Reverse Proxy

1. **Create Nginx configuration for Logto:**
   ```bash
   nano /etc/nginx/sites-available/logto
   ```
   
   Add this configuration:
   ```nginx
   # Main application
   server {
       listen 80;
       server_name auth.yourdomain.com;
       
       location / {
           proxy_pass http://127.0.0.1:3001;
           proxy_http_version 1.1;
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection 'upgrade';
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
           proxy_cache_bypass $http_upgrade;
       }
   }
   
   # Admin console
   server {
       listen 80;
       server_name admin.yourdomain.com;
       
       location / {
           proxy_pass http://127.0.0.1:3002;
           proxy_http_version 1.1;
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection 'upgrade';
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
           proxy_cache_bypass $http_upgrade;
       }
   }
   ```

2. **Enable the site and restart Nginx:**
   ```bash
   ln -s /etc/nginx/sites-available/logto /etc/nginx/sites-enabled/
   nginx -t
   systemctl restart nginx
   ```

3. **Set up SSL with Let's Encrypt:**
   ```bash
   certbot --nginx -d auth.yourdomain.com -d admin.yourdomain.com
   ```

## Step 5: Verify Your Custom Changes

1. **Test that numeric usernames work:**
   ```bash
   # Access your Logto instance
   curl https://auth.yourdomain.com/api/status
   
   # Try the admin console
   # Navigate to https://admin.yourdomain.com in your browser
   ```

2. **Create a test user with numeric username:**
   - Go to admin console
   - Create a new user with username "123456" or "888888"
   - It should work without any "username should not start with number" errors

## Step 6: Maintenance and Updates

1. **To update your code with new changes:**
   ```bash
   cd /opt/logto-koptnb
   git pull origin master
   docker compose -f docker-compose.prod.yml build
   docker compose -f docker-compose.prod.yml down
   docker compose -f docker-compose.prod.yml up -d
   ```

2. **View logs:**
   ```bash
   # All logs
   docker compose -f docker-compose.prod.yml logs -f
   
   # Just Logto app logs
   docker logs logto-app -f
   
   # Database logs
   docker logs logto-db -f
   ```

3. **Backup database:**
   ```bash
   # Create backup
   docker exec logto-db pg_dump -U logto logto > backup_$(date +%Y%m%d).sql
   
   # Restore backup
   cat backup_20240101.sql | docker exec -i logto-db psql -U logto logto
   ```

## Troubleshooting

### If build fails on VPS:
```bash
# Check Docker logs
docker compose -f docker-compose.prod.yml logs

# Rebuild without cache
docker compose -f docker-compose.prod.yml build --no-cache

# Check available disk space
df -h

# Check memory
free -m
```

### If numeric usernames still don't work:
1. Verify your changes are in the repository:
   ```bash
   cat packages/toolkit/core-kit/src/regex.ts | grep usernameRegEx
   # Should show: export const usernameRegEx = /^\w+$/;
   ```

2. Force rebuild:
   ```bash
   docker compose -f docker-compose.prod.yml down
   docker system prune -a
   docker compose -f docker-compose.prod.yml build --no-cache
   docker compose -f docker-compose.prod.yml up -d
   ```

## Security Recommendations

1. **Use strong passwords** for PostgreSQL
2. **Enable firewall** (ufw or iptables)
3. **Regular updates** of system packages
4. **Monitor logs** for suspicious activity
5. **Set up fail2ban** for brute force protection
6. **Regular backups** of database and configurations

## Your Custom Changes Summary

The Docker build will include your modifications:
- ✅ `packages/toolkit/core-kit/src/regex.ts` - Changed to `/^\w+$/` to allow numeric start
- ✅ `packages/experience/src/utils/form.ts` - Removed number start validation
- ✅ Users can now have fully numeric usernames like "123456", "999", etc.