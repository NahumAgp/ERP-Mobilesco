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

Antes de subir al servidor:

1. Copia `.env.example` a `.env`.
2. Define `JWT_SECRET_BASE64` con un secreto real en Base64.
3. Ajusta `MYSQL_ROOT_PASSWORD`, `MYSQL_DATABASE`, `MYSQL_USER` y `MYSQL_PASSWORD` si tu servidor usa otros valores.
4. Coloca los certificados en `./ssl` con estos nombres:
   - `fullchain.pem`
   - `privkey.pem`
5. Si tus certificados están en otra ruta, ajusta `SSL_CERTS_DIR`.

Levanta el stack:

```powershell
docker compose -f docker-compose.prod.yml up -d --build
```

Servicios:

- Frontend/Nginx: puertos 80 y 443
- Backend: red interna Docker
- MySQL: red interna Docker

## Comandos utiles

```powershell
docker compose --env-file .env.dev -f docker-compose.dev.yml ps
docker logs --tail 120 mobilesco-backend-dev
docker logs --tail 120 mobilesco-frontend-dev
docker compose --env-file .env.dev -f docker-compose.dev.yml down
docker compose -f docker-compose.prod.yml ps
docker logs --tail 120 mobilesco-backend
docker logs --tail 120 mobilesco-frontend
```

Los archivos subidos se montan en `./uploads` y se sirven desde `/uploads`.
