ardar # Docker

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

1. Copia `.env.example` a `.env` y cambia secretos, passwords y certificados.
2. Coloca los certificados en `./ssl` o ajusta `SSL_CERTS_DIR`.
3. Levanta:

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
```

Los archivos subidos se montan en `./uploads` y se sirven desde `/uploads`.
