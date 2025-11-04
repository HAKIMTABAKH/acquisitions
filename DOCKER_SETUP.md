# Docker Setup with Neon Database

This guide explains how to run the acquisitions application using Docker with Neon Database support for both development and production environments.

## 📋 Table of Contents

- [Prerequisites](#prerequisites)
- [Environment Setup](#environment-setup)
- [Development Environment](#development-environment)
- [Production Environment](#production-environment)
- [Troubleshooting](#troubleshooting)

## Prerequisites

1. **Docker & Docker Compose** installed
   - [Download Docker Desktop](https://www.docker.com/products/docker-desktop)
   
2. **Neon Account** (for both dev and prod)
   - Sign up at [neon.tech](https://neon.tech)
   - Create a project in the Neon Console

3. **Required Neon Credentials**
   - `NEON_API_KEY`: Get from [Neon Console → Account Settings → API Keys](https://console.neon.tech/app/settings/api-keys)
   - `NEON_PROJECT_ID`: Found in Project Settings → General
   - `PARENT_BRANCH_ID`: Your main branch ID (usually shown in Console)

---

## Environment Setup

### 1. Configure Environment Variables for Development

Copy the development environment template:

```bash
cp .env.development .env.dev.local
```

Edit `.env.dev.local` with your actual Neon credentials:

```env
PORT=3000
NODE_ENV=development
LOG_LEVEL=debug

DATABASE_URL=postgres://neon:npg@db:5432/neondb?sslmode=require

# Replace with your actual values
NEON_API_KEY=your_actual_neon_api_key
NEON_PROJECT_ID=your_actual_project_id
PARENT_BRANCH_ID=your_actual_branch_id
```

### 2. Configure Environment Variables for Production

Copy the production environment template:

```bash
cp .env.production .env.prod.local
```

Edit `.env.prod.local` with your Neon Cloud connection string:

```env
PORT=3000
NODE_ENV=production
LOG_LEVEL=info

# Your actual Neon Cloud database URL
DATABASE_URL=postgresql://user:password@ep-xyz-123.c-2.eu-central-1.aws.neon.tech/dbname?sslmode=require
```

### 3. Update .gitignore

Add these lines to your `.gitignore`:

```
.env.dev.local
.env.prod.local
.neon_local/
```

---

## Development Environment

The development setup uses **Neon Local**, a Docker-based proxy that creates ephemeral database branches.

### Architecture

```
┌─────────────────┐
│   Your App      │
│  (Docker)       │
└────────┬────────┘
         │
         │ DATABASE_URL=postgres://neon:npg@db:5432/neondb
         │
┌────────▼────────┐
│  Neon Local     │
│  Proxy          │
│  (Docker)       │
└────────┬────────┘
         │
         │ API Connection
         │
┌────────▼────────┐
│  Neon Cloud     │
│  (Ephemeral     │
│   Branch)       │
└─────────────────┘
```

### Start Development Environment

```bash
# Using the .env.dev.local file
docker-compose --env-file .env.dev.local -f docker-compose.dev.yml up --build
```

Or create a shortcut by adding to `package.json`:

```json
"scripts": {
  "docker:dev": "docker-compose --env-file .env.dev.local -f docker-compose.dev.yml up --build",
  "docker:dev:down": "docker-compose -f docker-compose.dev.yml down"
}
```

Then run:

```bash
npm run docker:dev
```

### What Happens in Development Mode

1. **Neon Local** container starts and connects to Neon Cloud
2. Creates an **ephemeral branch** from your `PARENT_BRANCH_ID`
3. Your app connects to `db:5432` (Neon Local proxy)
4. Neon Local routes all queries to the ephemeral branch
5. When you stop the container, the ephemeral branch is **automatically deleted**

### Features

✅ **Fresh database on every startup** - No manual cleanup needed  
✅ **Isolated from production** - Safe to test destructive operations  
✅ **Fast branch creation** - Branches are created in seconds  
✅ **Automatic cleanup** - Branches deleted when container stops  

### Development Tips

#### Enable Hot Reloading

The `docker-compose.dev.yml` already mounts your `src/` directory as a volume, so changes to your code will be reflected if you use nodemon or a similar tool.

To add nodemon support:

```bash
npm install --save-dev nodemon
```

Update `package.json`:

```json
"scripts": {
  "dev": "nodemon --watch src src/index.js"
}
```

Update `docker-compose.dev.yml` CMD:

```yaml
app:
  # ... other config
  command: npm run dev
```

#### Persist Branches Between Restarts

To keep the same branch between container restarts, set `DELETE_BRANCH: "false"` in `docker-compose.dev.yml`:

```yaml
db:
  environment:
    DELETE_BRANCH: "false"
```

#### Run Database Migrations

```bash
# From inside the app container
docker exec -it acquisitions-app-dev npm run db:migrate

# Or from host machine
docker-compose -f docker-compose.dev.yml exec app npm run db:migrate
```

---

## Production Environment

The production setup connects directly to **Neon Cloud** without using Neon Local.

### Architecture

```
┌─────────────────┐
│   Your App      │
│  (Docker)       │
└────────┬────────┘
         │
         │ DATABASE_URL (direct connection)
         │
┌────────▼────────┐
│  Neon Cloud     │
│  (Production    │
│   Branch)       │
└─────────────────┘
```

### Deploy to Production

#### Option 1: Using Docker Compose

```bash
# Load production environment and start
docker-compose --env-file .env.prod.local -f docker-compose.prod.yml up -d --build
```

#### Option 2: Using Environment Variables (Recommended for CI/CD)

```bash
# Set environment variable
export DATABASE_URL="postgresql://user:password@ep-xyz.neon.tech/dbname?sslmode=require"

# Start production stack
docker-compose -f docker-compose.prod.yml up -d --build
```

#### Option 3: Build and Push to Registry

```bash
# Build image
docker build -t your-registry/acquisitions:latest .

# Push to registry
docker push your-registry/acquisitions:latest

# Run on production server
docker run -d \
  -p 3000:3000 \
  -e NODE_ENV=production \
  -e DATABASE_URL=$DATABASE_URL \
  --name acquisitions-app \
  your-registry/acquisitions:latest
```

### Production Checklist

- [ ] Use actual Neon Cloud connection string (not Neon Local)
- [ ] Store `DATABASE_URL` in secrets manager (e.g., AWS Secrets Manager, Azure Key Vault)
- [ ] Enable connection pooling in Neon (use `-pooler` endpoint)
- [ ] Set appropriate resource limits in Docker Compose or orchestrator
- [ ] Configure proper logging and monitoring
- [ ] Run database migrations before deploying new version
- [ ] Set up health checks and restart policies

---

## Database Connection Strings

### Development (Neon Local)

```
postgres://neon:npg@db:5432/neondb?sslmode=require
```

- **Host**: `db` (service name in docker-compose)
- **Port**: `5432`
- **User**: `neon` (default for Neon Local)
- **Password**: `npg` (default for Neon Local)
- **Database**: `neondb` (or your database name)

### Production (Neon Cloud)

```
postgresql://user:password@ep-xyz-123.c-2.eu-central-1.aws.neon.tech/dbname?sslmode=require
```

Use the connection string from your Neon Console:
- Navigate to your project
- Select your production branch
- Copy the connection string under "Connection Details"

**⚠️ Important**: Use the **pooled connection string** (with `-pooler` subdomain) for production applications to avoid connection limits.

---

## Troubleshooting

### Issue: "Connection refused" when app tries to connect to database

**Solution**: Ensure the database service is healthy before the app starts. The `docker-compose.dev.yml` already includes a health check and `depends_on` condition.

### Issue: Neon Local fails to create branch

**Possible causes**:
1. Invalid `NEON_API_KEY`, `NEON_PROJECT_ID`, or `PARENT_BRANCH_ID`
2. Network connectivity issues

**Solution**: 
- Verify your Neon credentials in the console
- Check Docker logs: `docker logs acquisitions-neon-local`

### Issue: "self-signed certificate" error with @neondatabase/serverless driver

If using the Neon serverless driver, you need additional configuration for Neon Local. Update your database connection code:

```javascript
import { neon, neonConfig } from '@neondatabase/serverless';

// Only for development with Neon Local
if (process.env.NODE_ENV === 'development') {
  neonConfig.fetchEndpoint = 'http://db:5432/sql';
  neonConfig.useSecureWebSocket = false;
  neonConfig.poolQueryViaFetch = true;
}

const sql = neon(process.env.DATABASE_URL);
```

### Issue: App can't connect on Windows with Docker Desktop

**Solution**: Ensure Docker Desktop is using **gRPC FUSE** instead of VirtioFS:
1. Open Docker Desktop Settings
2. Go to General
3. Under "Choose file sharing implementation for your containers", select "gRPC FUSE"
4. Restart Docker Desktop

### Issue: Permission denied when mounting volumes on Windows

**Solution**: Ensure the project directory is in a shared drive:
1. Docker Desktop → Settings → Resources → File Sharing
2. Add your project directory path
3. Apply & Restart

---

## Additional Resources

- [Neon Local Documentation](https://neon.com/docs/local/neon-local)
- [Neon Connection Documentation](https://neon.com/docs/connect/connect-from-any-app)
- [Docker Compose Documentation](https://docs.docker.com/compose/)

---

## Quick Reference Commands

```bash
# Development
npm run docker:dev                    # Start dev environment
npm run docker:dev:down              # Stop dev environment
docker logs acquisitions-neon-local  # View Neon Local logs
docker logs acquisitions-app-dev     # View app logs

# Production
docker-compose -f docker-compose.prod.yml up -d    # Start production
docker-compose -f docker-compose.prod.yml down     # Stop production
docker-compose -f docker-compose.prod.yml logs -f  # View logs

# Database Operations
docker exec -it acquisitions-app-dev npm run db:migrate   # Run migrations
docker exec -it acquisitions-app-dev npm run db:studio    # Open Drizzle Studio
```

---

**Note**: Always ensure your `.env.dev.local` and `.env.prod.local` files are **never committed** to version control. Add them to `.gitignore`.
