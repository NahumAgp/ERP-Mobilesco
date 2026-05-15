# Docker

## Desarrollo

Usa el archivo `.env.dev` incluido para levantar MySQL, backend Spring Boot y frontend Nginx:

```powershell
docker compose --env-file .env.dev -f docker-compose.dev.yml up -d --build
```

Servicios:

- Frontend: http://localhost
- Backend publicado: http://localhost:8082
- MySQL publicado: localhost:3307
- Swagger en dev: http://localhost:8082/swagger-ui/index.html

## Produccion

En el VPS usamos **Swarm + Traefik**. Traefik ya ocupa `80/443`, así que el stack de Mobilesco no debe publicar esos puertos directamente.

Antes de desplegar:

1. Copia `.env.example` a `.env`.
2. Define `JWT_SECRET_BASE64` con un secreto real en Base64.
3. Ajusta `MYSQL_ROOT_PASSWORD`, `MYSQL_DATABASE`, `MYSQL_USER` y `MYSQL_PASSWORD` si tu servidor usa otros valores.
4. Define `TRAEFIK_NETWORK` con el nombre real de la red de Traefik del VPS.
   - Por defecto usamos `MobilescoNet`, que es la red que ya tienes en Portainer.
5. Si no sabes el nombre, ejecútalo en el servidor:

```powershell
docker network ls
```

6. Si el volumen de MySQL ya existe, déjalo tal cual. El stack lo reutiliza como `mobilesco_mysql_data`.

Despliegue:

```powershell
docker build -t mobilesco-backend:latest ./mobilesco-back
docker build -t mobilesco-frontend:latest ./mobilesco-front
docker stack deploy -c docker-stack.yml mobilesco
```

Actualización típica:

```powershell
git pull origin main
git submodule update --init --recursive
docker build -t mobilesco-backend:latest ./mobilesco-back
docker build -t mobilesco-frontend:latest ./mobilesco-front
docker stack deploy -c docker-stack.yml mobilesco
```

Servicios:

- Frontend: expuesto por Traefik
- Backend: red interna Swarm
- MySQL: red interna Swarm

## Comandos utiles

```powershell
docker compose --env-file .env.dev -f docker-compose.dev.yml ps
docker logs --tail 120 mobilesco-backend-dev
docker logs --tail 120 mobilesco-frontend-dev
docker compose --env-file .env.dev -f docker-compose.dev.yml down
docker compose -f docker-compose.prod.yml ps
docker logs --tail 120 mobilesco-backend
docker logs --tail 120 mobilesco-frontend
docker stack ls
docker service ls | findstr mobilesco
docker service logs mobilesco_backend
docker service logs mobilesco_frontend
```

Los archivos subidos se montan en `./uploads` y se sirven desde `/uploads`.
