# Docker

## Desarrollo local

Usa el archivo `.env.dev` incluido para levantar MySQL, backend Spring Boot y frontend Nginx:

```powershell
docker compose --env-file .env.dev -f docker-compose.dev.yml up -d --build
```

Servicios:

- Frontend: http://localhost
- Backend publicado: http://localhost:8082
- MySQL publicado: localhost:3307
- Swagger en dev: http://localhost:8082/swagger-ui/index.html

Comandos utiles:

```powershell
docker compose --env-file .env.dev -f docker-compose.dev.yml ps
docker logs --tail 120 mobilesco-backend-dev
docker logs --tail 120 mobilesco-frontend-dev
docker compose --env-file .env.dev -f docker-compose.dev.yml down
```

## Atajo: dokeriza local

Cuando pida `dokeriza local`, significa probar el proyecto en el entorno seguro de Docker Desktop, reconstruyendo las imagenes locales y levantando el compose de desarrollo.

Ejecutar desde la raiz del repo:

```powershell
docker compose --env-file .env.dev -f docker-compose.dev.yml up -d --build
docker compose --env-file .env.dev -f docker-compose.dev.yml ps
docker logs --tail 120 mobilesco-backend-dev
docker logs --tail 120 mobilesco-frontend-dev
```

Si hace falta limpiar el entorno local antes de reconstruir:

```powershell
docker compose --env-file .env.dev -f docker-compose.dev.yml down
docker compose --env-file .env.dev -f docker-compose.dev.yml up -d --build
```

## Produccion con SetupOrion, Traefik y Portainer

En el VPS se usa Docker Swarm con Traefik y Portainer:

- Traefik publica los puertos `80` y `443`.
- Portainer queda en `https://portainer.mobilesco.cloud`.
- El ERP queda en `https://erp.mobilesco.cloud`.
- Tambien se aceptan `https://mobilesco.cloud` y `https://www.mobilesco.cloud`.
- La red compartida de Traefik es `MobilescoNet`.
- El stack del ERP se llama `mobilesco`.
- La carpeta recomendada del ERP en el VPS es `/opt/mobilesco`.
- El archivo de despliegue del ERP es `docker-stack.yml`.

Regla principal: solo Traefik debe publicar `80` y `443`. El frontend del ERP no debe usar `ports: "80:80"` ni `ports: "443:443"` en produccion con SetupOrion.

## Variables de entorno en el VPS

El archivo `/opt/mobilesco/.env` debe tener una variable por linea. Usa comillas simples cuando el valor tenga caracteres especiales como `&`, `+`, `/` o `=`.

Ejemplo:

```env
MYSQL_ROOT_PASSWORD='clave_root_mysql'
MYSQL_DATABASE=mobilesco
MYSQL_USER=mobilesco_user
MYSQL_PASSWORD='clave_usuario_mysql'
SPRING_PROFILE=prod
SPRING_DATASOURCE_URL='jdbc:mysql://db:3306/mobilesco?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC'
JWT_SECRET_BASE64='secreto_jwt_base64_largo'
TRAEFIK_NETWORK=MobilescoNet
```

No partas passwords o secretos en varias lineas.

Para generar secretos:

```bash
openssl rand -base64 48
```

Para cargar el `.env` antes de desplegar:

```bash
cd /opt/mobilesco
set -a
source .env
set +a
```

Si quieres comprobar que el secreto JWT quedo cargado en la shell:

```bash
echo "$JWT_SECRET_BASE64"
```

## Primer despliegue en Swarm

Verifica que exista la red externa de Traefik:

```bash
docker network ls
```

Si `MobilescoNet` no existe, revisa en Portainer el nombre real de la red de Traefik y ajusta `TRAEFIK_NETWORK` en `.env`.

Prepara imagenes y despliega:

```bash
cd /opt/mobilesco
git pull
git submodule update --init --recursive
set -a
source .env
set +a
docker build -t mobilesco-backend:latest ./mobilesco-back
docker build -t mobilesco-frontend:latest ./mobilesco-front
docker stack deploy -c docker-stack.yml mobilesco
docker stack services mobilesco
```

Notas:

- El volumen externo de MySQL es `mobilesco_mysql_data`; no lo borres si ya tiene datos reales.
- El stack crea el volumen de uploads como `mobilesco_uploads_data`.
- La red interna del stack se llama `mobilesco_overlay`.
- En Swarm no uses mounts relativos como `./uploads`.

## Actualizar desde Git

Cuando los cambios ya estan subidos al repo:

```bash
cd /opt/mobilesco
git pull
git submodule update --init --recursive
```

El `git pull` actualiza el repo principal. El `git submodule update` actualiza `mobilesco-back` y `mobilesco-front` si el cambio vive dentro de esos submodulos.

## Actualizar solo backend

Usa esto cuando cambias Java, Spring, CORS, seguridad, entidades, servicios, controladores o configuracion del backend:

```bash
cd /opt/mobilesco
git pull
git submodule update --init --recursive
set -a
source .env
set +a
docker build -t mobilesco-backend:latest ./mobilesco-back
docker service update --force mobilesco_backend
docker service logs mobilesco_backend --since 2m
```

Debe aparecer algo como:

```txt
Started MobilescoBackApplication
Tomcat started on port 8081
```

## Actualizar solo frontend

Usa esto cuando cambias React, estilos, assets, pantallas, rutas del frontend o `nginx.conf`:

```bash
cd /opt/mobilesco
git pull
git submodule update --init --recursive
set -a
source .env
set +a
docker build -t mobilesco-frontend:latest ./mobilesco-front
docker service update --force mobilesco_frontend
docker service logs mobilesco_frontend --since 2m
```

Despues recarga el navegador con `Ctrl + F5`.

## Actualizar backend y frontend

Usa esto cuando cambias ambos lados:

```bash
cd /opt/mobilesco
git pull
git submodule update --init --recursive
set -a
source .env
set +a
docker build -t mobilesco-backend:latest ./mobilesco-back
docker build -t mobilesco-frontend:latest ./mobilesco-front
docker service update --force mobilesco_backend
docker service update --force mobilesco_frontend
docker stack services mobilesco
```

## Redeplegar todo el stack

Usa esto cuando cambias `docker-stack.yml`, variables de entorno, volumenes, redes o labels de Traefik:

```bash
cd /opt/mobilesco
set -a
source .env
set +a
docker stack deploy -c docker-stack.yml mobilesco
docker stack services mobilesco
```

## Base de datos

Ver estado de MySQL:

```bash
docker service ps mobilesco_db
docker service logs mobilesco_db --tail 80
```

Entrar al contenedor de MySQL:

```bash
docker ps --filter name=mobilesco_db
docker exec -it ID_DEL_CONTENEDOR mysql -u root -p
```

Dentro de MySQL:

```sql
SHOW DATABASES;
USE mobilesco;
SHOW TABLES;
```

Importante: no borres el volumen de MySQL si ya hay datos reales.

Solo en instalacion nueva, si necesitas recrear la base desde cero:

```bash
docker stack rm mobilesco
docker volume rm mobilesco_mysql_data
set -a
source /opt/mobilesco/.env
set +a
docker stack deploy -c /opt/mobilesco/docker-stack.yml mobilesco
```

Esto elimina los datos de la base.

## Restaurar backup de base y uploads

Usa esta guia para restaurar un backup completo del ERP en el VPS. El backup principal debe contener:

- `database.sql`
- `uploads.tar.gz`

No uses el backup de configuracion sensible para restaurar datos, porque puede traer `.env` y certificados viejos.

Ejemplo de backup:

```txt
backups/mobilesco-backup-20260513-224322.tar.gz
```

Crear carpeta de backups en el VPS:

```bash
mkdir -p /opt/mobilesco/backups
```

Subir el backup desde Windows al VPS:

```powershell
scp "C:\Users\nahum\OneDrive\Documentos\ERP-Mobilesco-Nahum\ERP-Mobilesco\backups\mobilesco-backup-20260513-224322.tar.gz" root@76.13.30.221:/opt/mobilesco/backups/
```

Extraer el backup en el VPS:

```bash
cd /opt/mobilesco/backups
tar -xzf mobilesco-backup-20260513-224322.tar.gz
cd mobilesco-backup-20260513-224322
ls -la
```

Antes de restaurar, pausar backend y frontend:

```bash
docker service scale mobilesco_backend=0
docker service scale mobilesco_frontend=0
```

Cargar variables del `.env`:

```bash
cd /opt/mobilesco
set -a
source .env
set +a
DB_CONTAINER=$(docker ps -q --filter name=mobilesco_db)
```

Recrear la base `mobilesco`:

```bash
docker exec -i -e MYSQL_PWD="$MYSQL_ROOT_PASSWORD" "$DB_CONTAINER" mysql -u root -e "DROP DATABASE IF EXISTS mobilesco; CREATE DATABASE mobilesco CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci; GRANT ALL PRIVILEGES ON mobilesco.* TO '$MYSQL_USER'@'%'; FLUSH PRIVILEGES;"
```

Importar `database.sql`:

```bash
docker exec -i -e MYSQL_PWD="$MYSQL_ROOT_PASSWORD" "$DB_CONTAINER" mysql -u root mobilesco < /opt/mobilesco/backups/mobilesco-backup-20260513-224322/database.sql
```

Restaurar uploads:

```bash
docker run --rm -v mobilesco_uploads_data:/target -v /opt/mobilesco/backups/mobilesco-backup-20260513-224322:/backup alpine sh -c "rm -rf /target/* && tar -xzf /backup/uploads.tar.gz -C /target --strip-components=1"
```

Levantar backend y frontend:

```bash
docker service scale mobilesco_backend=1
docker service scale mobilesco_frontend=1
```

Verificar:

```bash
docker stack services mobilesco
docker service logs mobilesco_backend --since 2m
docker service logs mobilesco_frontend --since 2m
curl -I https://erp.mobilesco.cloud
```

Advertencia: este proceso reemplaza la base actual y los uploads actuales por los del backup. No lo ejecutes sobre datos reales sin confirmar antes.

## Comandos utiles en produccion

Ver stacks:

```bash
docker stack ls
```

Ver servicios del ERP:

```bash
docker stack services mobilesco
```

Ver tareas de cada servicio:

```bash
docker service ps mobilesco_db
docker service ps mobilesco_backend
docker service ps mobilesco_frontend
```

Ver logs recientes:

```bash
docker service logs mobilesco_db --since 5m
docker service logs mobilesco_backend --since 5m
docker service logs mobilesco_frontend --since 5m
```

Ver variables aplicadas en un servicio:

```bash
docker service inspect mobilesco_backend --format '{{range .Spec.TaskTemplate.ContainerSpec.Env}}{{println .}}{{end}}'
docker service inspect mobilesco_db --format '{{range .Spec.TaskTemplate.ContainerSpec.Env}}{{println .}}{{end}}'
```

Ver certificados HTTPS:

```bash
echo | openssl s_client -servername erp.mobilesco.cloud -connect erp.mobilesco.cloud:443 2>/dev/null | openssl x509 -noout -issuer -subject -dates
echo | openssl s_client -servername portainer.mobilesco.cloud -connect portainer.mobilesco.cloud:443 2>/dev/null | openssl x509 -noout -issuer -subject -dates
```

Debe aparecer `Let's Encrypt`.

Probar respuesta del ERP:

```bash
curl -I https://erp.mobilesco.cloud
```

Ver Traefik y Portainer:

```bash
docker stack services traefik
docker stack services portainer
docker service logs traefik_traefik --tail 80
docker service logs portainer_portainer --tail 80
```

Si Traefik escribe logs a archivo:

```bash
docker ps --filter name=traefik_traefik
docker exec ID_DEL_CONTENEDOR sh -c "tail -120 /var/log/traefik/traefik.log"
```

## Archivos subidos

Los archivos subidos por el backend se guardan en el volumen Docker `mobilesco_uploads_data` y se sirven desde `/uploads`.
