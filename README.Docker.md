# Docker Setup for Kyood Application

This document provides instructions for running the Kyood application using Docker.

## Prerequisites

- Docker (version 20.10 or higher)
- Docker Compose (version 2.0 or higher)

## Architecture

The application consists of the following services:

- **Frontend**: Next.js application (port 3000)
- **Backend**: Django application with Channels for WebSocket support (port 8000)
- **Database**: PostgreSQL 14 (port 5432)
- **Redis**: For Celery task queue and Django Channels (port 6379)
- **Celery Worker**: Background task processor
- **Celery Beat**: Periodic task scheduler

## Quick Start

1. **Clone the repository** (if not already done)

2. **Create environment file** (optional)
   ```bash
   cp .env.example .env
   # Edit .env with your preferred values
   ```

3. **Build and start all services**
   ```bash
   docker-compose up --build
   ```

   Or run in detached mode:
   ```bash
   docker-compose up -d --build
   ```

4. **Run database migrations**
   ```bash
   docker-compose exec backend python manage.py migrate
   ```

5. **Create a superuser** (optional)
   ```bash
   docker-compose exec backend python manage.py createsuperuser
   ```

6. **Access the application**
   - Frontend: http://localhost:3000
   - Backend API: http://localhost:8000
   - Django Admin: http://localhost:8000/admin

## Common Commands

### View logs
```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f backend
docker-compose logs -f frontend
docker-compose logs -f celery_worker
```

### Stop services
```bash
docker-compose stop
```

### Stop and remove containers
```bash
docker-compose down
```

### Stop and remove containers with volumes (⚠️ deletes data)
```bash
docker-compose down -v
```

### Rebuild a specific service
```bash
docker-compose up -d --build backend
```

### Execute commands in containers
```bash
# Django shell
docker-compose exec backend python manage.py shell

# Create migrations
docker-compose exec backend python manage.py makemigrations

# Run migrations
docker-compose exec backend python manage.py migrate

# Collect static files
docker-compose exec backend python manage.py collectstatic --noinput
```

### Access container shell
```bash
# Backend
docker-compose exec backend sh

# Frontend
docker-compose exec frontend sh
```

## Development vs Production

### Development Mode
The current setup is configured for development with:
- Hot reloading enabled via volume mounts
- Debug mode enabled
- SQLite can be used (default in settings.py)

### Production Considerations
For production deployment, you should:

1. **Update environment variables**
   - Set `DEBUG=0`
   - Use a strong `SECRET_KEY`
   - Configure proper `ALLOWED_HOSTS`
   - Use PostgreSQL instead of SQLite

2. **Update Django settings**
   - Modify `campaign_backend/backend/core/settings.py` to read from environment variables:
   ```python
   import os
   
   DEBUG = os.getenv('DEBUG', 'False') == '1'
   SECRET_KEY = os.getenv('SECRET_KEY', 'fallback-secret-key')
   ALLOWED_HOSTS = os.getenv('ALLOWED_HOSTS', 'localhost').split(',')
   
   DATABASES = {
       'default': {
           'ENGINE': os.getenv('DATABASE_ENGINE', 'django.db.backends.sqlite3'),
           'NAME': os.getenv('DATABASE_NAME', BASE_DIR / 'db.sqlite3'),
           'USER': os.getenv('DATABASE_USER', ''),
           'PASSWORD': os.getenv('DATABASE_PASSWORD', ''),
           'HOST': os.getenv('DATABASE_HOST', ''),
           'PORT': os.getenv('DATABASE_PORT', ''),
       }
   }
   
   CHANNEL_LAYERS = {
       "default": {
           "BACKEND": "channels_redis.core.RedisChannelLayer",
           "CONFIG": {
               "hosts": [(os.getenv('REDIS_HOST', '127.0.0.1'), int(os.getenv('REDIS_PORT', 6379)))],
           },
       },
   }
   
   CELERY_BROKER_URL = os.getenv('CELERY_BROKER_URL', 'redis://localhost:6379/0')
   ```

3. **Use production-grade web server**
   - The Dockerfile already uses Daphne for ASGI support
   - Consider using Nginx as a reverse proxy

4. **Enable HTTPS**
   - Configure SSL certificates
   - Update CORS settings

5. **Optimize Docker images**
   - Remove development dependencies
   - Use multi-stage builds (already implemented for frontend)

## Troubleshooting

### Port already in use
If you get a port conflict error, either:
- Stop the service using that port
- Change the port mapping in `docker-compose.yml`

### Database connection errors
- Ensure the database service is healthy: `docker-compose ps`
- Check database logs: `docker-compose logs db`
- Verify environment variables are correct

### Frontend can't connect to backend
- Check that `NEXT_PUBLIC_API_URL` is set correctly
- Ensure backend is running: `docker-compose ps backend`
- Check backend logs: `docker-compose logs backend`

### Celery tasks not running
- Check celery worker logs: `docker-compose logs celery_worker`
- Verify Redis is running: `docker-compose ps redis`
- Check Celery Beat logs: `docker-compose logs celery_beat`

### Permission issues
If you encounter permission issues with volumes:
```bash
# Fix ownership (Linux/Mac)
sudo chown -R $USER:$USER ./campaign_backend/backend/media
```

## Volumes

The setup uses Docker volumes for persistent data:
- `postgres_data`: PostgreSQL database files
- `media_data`: Django media files (uploads)

To backup volumes:
```bash
# Backup database
docker-compose exec db pg_dump -U kyood_user kyood > backup.sql

# Restore database
docker-compose exec -T db psql -U kyood_user kyood < backup.sql
```

## Network

All services are connected via a default Docker network, allowing them to communicate using service names as hostnames (e.g., `backend`, `db`, `redis`).

## Additional Notes

- The backend uses Daphne ASGI server for WebSocket support
- Static files are collected during the Docker build process
- The frontend uses Next.js standalone output for optimized Docker images
- Development changes are reflected immediately due to volume mounts
