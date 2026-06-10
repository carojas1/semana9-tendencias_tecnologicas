# Informe de Práctica: Contenerización de Frontend React con Docker

---

## 1. Título

Despliegue Full-Stack Contenerizado: Backend Spring Boot (Kotlin), Base de Datos PostgreSQL, Panel pgAdmin y Frontend React con Nginx usando Docker Compose y Multi-Stage Builds

---

## 2. Tiempo de Duración

120 minutos

---

## 3. Fundamentos

En esta práctica amplié la infraestructura backend de la semana anterior agregando un contenedor de frontend desarrollado con React y Vite. Aprendí cómo contenerizar una aplicación frontend usando Docker con Multi-Stage Build y cómo configurar nginx como proxy inverso para que el frontend pueda comunicarse con el backend sin problemas de CORS.

Docker nos permite crear contenedores que funcionan igual en cualquier ambiente, ya sea desarrollo o producción. Esto es muy útil porque evita el problema clásico de "en mi máquina funciona pero en la del profe no".

Para manejar los cuatro servicios al mismo tiempo usé Docker Compose, que permite definir todos los contenedores en un solo archivo YAML y levantarlos con un comando. Esto facilita mucho la administración porque todo queda documentado en un solo lugar (Docker Inc., 2024b).

Los cuatro servicios que implementé fueron:

- **PostgreSQL**: motor de base de datos relacional donde se guarda la información de los usuarios.
- **pgAdmin**: herramienta web para administrar PostgreSQL de forma visual desde el navegador.
- **Spring Boot Backend**: aplicación principal en Kotlin que expone una API REST con los datos de los usuarios.
- **Frontend React**: interfaz de usuario que consume la API y muestra la tabla de usuarios.

También apliqué los **Multi-Stage Builds** (Docker Inc., 2024a). En el frontend, la primera etapa usa Node.js para compilar el proyecto con Vite, y la segunda etapa usa nginx para servir los archivos estáticos. Gracias a esto la imagen final quedó mucho más liviana sin tener Node ni npm en producción.

El concepto más importante que aprendí es el **proxy inverso con nginx**. Al configurar nginx para redirigir las peticiones `/api/` hacia el contenedor del backend, el frontend puede hacer llamadas a rutas relativas como `/api/users` en lugar de hardcodear `http://localhost:8081`. Esto funciona porque los contenedores en la misma red Docker se comunican usando el nombre del servicio como hostname.

Otros conceptos aplicados:

- Redes privadas Docker (`backend_network`) para comunicación interna entre contenedores.
- Volúmenes persistentes para que los datos de PostgreSQL y pgAdmin no se pierdan al reiniciar.
- Archivo `.env` para centralizar credenciales de forma segura.
- `healthcheck` en PostgreSQL para que el backend espere a que la BD esté lista antes de conectarse.
- `depends_on` en Docker Compose para controlar el orden de arranque de los contenedores.
- Flyway para las migraciones automáticas de la base de datos al iniciar el backend.

---

## 4. Conocimientos Previos

Para realizar esta práctica necesité saber sobre:

- Comandos básicos de terminal (PowerShell).
- Docker: imágenes, contenedores, volúmenes y redes.
- Docker Compose para orquestar múltiples servicios.
- Variables de entorno y archivos `.env`.
- Base de datos PostgreSQL y lenguaje SQL básico.
- Aplicaciones Spring Boot con Kotlin y JPA.
- React 18 y Vite para el desarrollo frontend.
- JavaScript (fetch API, hooks de React).
- Nginx: configuración básica de servidor y proxy inverso.
- Conceptos de redes: puertos, DNS interno de Docker.
- Migraciones con Flyway.

---

## 5. Objetivos a Alcanzar

### Objetivo General

Implementar una infraestructura full-stack completa usando Docker Compose con PostgreSQL, pgAdmin, Spring Boot y React, aplicando Multi-Stage Build para optimizar las imágenes y nginx como proxy inverso.

### Objetivos Específicos

- Crear un proyecto React con Vite que consuma la API REST del backend.
- Escribir un Dockerfile con Multi-Stage Build para el frontend.
- Configurar nginx como servidor estático y proxy inverso hacia el backend.
- Integrar el contenedor frontend al `docker-compose.yml` existente.
- Verificar la comunicación entre todos los contenedores a través de la red interna.
- Confirmar que la tabla de usuarios se muestra correctamente en el navegador.
- Confirmar que pgAdmin puede administrar la base de datos.

---

## 6. Equipo Necesario

- Computador con Windows 11.
- Docker Desktop instalado y en ejecución.
- Git.
- Visual Studio Code.
- Node.js 20 (para desarrollo local).
- Navegador web (Microsoft Edge).
- Terminal (PowerShell).
- Conexión a internet para descargar las imágenes Docker.

---

## 7. Material de Apoyo

- Documentación oficial Docker: https://docs.docker.com/
- Documentación Docker Compose: https://docs.docker.com/compose/
- Repositorio base del proyecto: https://github.com/maguaman2/tendencias-mar22-security.git
- Documentación PostgreSQL 16: https://www.postgresql.org/docs/16/
- Spring Boot Documentation: https://spring.io/projects/spring-boot
- pgAdmin 4 Docs: https://www.pgadmin.org/docs/pgadmin4/latest/
- React Documentation: https://react.dev/
- Vite Documentation: https://vitejs.dev/
- Nginx Documentation: https://nginx.org/en/docs/

---

## 8. Procedimiento

### Paso 1: Configurar el archivo `.env`

El archivo `.env` centraliza todas las credenciales y variables de configuración para los servicios:

```env
# Base de Datos PostgreSQL
POSTGRES_DB=security_db
POSTGRES_USER=admin_user
POSTGRES_PASSWORD=741951
POSTGRES_PORT=5432

# pgAdmin
PGADMIN_DEFAULT_EMAIL=admin@security.com
PGADMIN_DEFAULT_PASSWORD=admin_password_123
PGADMIN_PORT=8090

# Backend Spring Boot
DB_SERVER=postgres_db
DB_PORT=5432
DB_NAME=security_db
DB_USER=admin_user
DB_PASSWORD=741951
SPRING_PROFILES_ACTIVE=prod
```

| Variable | Descripción |
|---|---|
| `POSTGRES_DB` | Nombre de la base de datos |
| `POSTGRES_USER` | Usuario de PostgreSQL |
| `POSTGRES_PASSWORD` | Contraseña de PostgreSQL |
| `PGADMIN_DEFAULT_EMAIL` | Email para entrar a pgAdmin |
| `PGADMIN_DEFAULT_PASSWORD` | Contraseña de pgAdmin |
| `DB_SERVER` | Nombre del contenedor BD (DNS interno Docker) |
| `SPRING_PROFILES_ACTIVE` | Perfil activo de Spring Boot |

---

### Paso 2: Revisar la entidad Usuario

La entidad `User` está mapeada a la tabla `users` de PostgreSQL con los campos `id`, `username`, `email`, `passwordHash`, `isActive` y `createdAt`.

**`User.kt` — Entidad JPA en Visual Studio Code:**

![User.kt - Entidad JPA en Kotlin](https://raw.githubusercontent.com/carojas1/semana9-tendencias_tecnologicas/assets/screenshots/vsc_userkt.png)

---

### Paso 3: Revisar el controlador REST

El `UserController` expone el endpoint `GET /users` que retorna todos los usuarios en formato JSON.

**`UserController.kt` en Visual Studio Code:**

![UserController.kt - controlador REST endpoint GET /users](https://raw.githubusercontent.com/carojas1/semana9-tendencias_tecnologicas/assets/screenshots/vsc_controller.png)

---

### Paso 4: Dockerfile del Backend con Multi-Stage Build

```dockerfile
# ETAPA 1: Compilacion con Maven
FROM maven:3.9.6-eclipse-temurin-21-alpine AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline -B
COPY src ./src
RUN mvn clean package -DskipTests

# ETAPA 2: Solo ejecucion con JRE liviano
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8081
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**`Dockerfile` del backend en Visual Studio Code:**

![Dockerfile del backend Spring Boot - multi-stage build](https://raw.githubusercontent.com/carojas1/semana9-tendencias_tecnologicas/assets/screenshots/vsc_dockerfile_backend.png)

La ventaja del Multi-Stage Build es que la imagen final solo contiene el JRE y el `.jar`, sin Maven ni el JDK completo, lo que la hace mucho más liviana.

---

### Paso 5: Crear el componente React (`App.jsx`)

El componente principal hace un `fetch` a `/api/users` al montar el componente y muestra los datos en una tabla HTML. La ruta es relativa porque nginx intercepta `/api/` y la redirige al backend.

**`App.jsx` en Visual Studio Code:**

![App.jsx - componente React con la tabla de usuarios](https://raw.githubusercontent.com/carojas1/semana9-tendencias_tecnologicas/assets/screenshots/vsc_appjsx.png)

---

### Paso 6: Dockerfile del Frontend con Multi-Stage Build

```dockerfile
# ETAPA 1: Compilar con Node
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# ETAPA 2: Servir con nginx
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

**`Dockerfile` del frontend en Visual Studio Code:**

![Dockerfile del frontend React - node builder + nginx alpine](https://raw.githubusercontent.com/carojas1/semana9-tendencias_tecnologicas/assets/screenshots/vsc_dockerfile_frontend.png)

---

### Paso 7: Configurar nginx como Proxy Inverso

```nginx
server {
    listen 80;

    location / {
        root /usr/share/nginx/html;
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://backend_app:8081/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**`nginx.conf` en Visual Studio Code:**

![nginx.conf con proxy inverso hacia backend_app:8081](https://raw.githubusercontent.com/carojas1/semana9-tendencias_tecnologicas/assets/screenshots/vsc_nginx.png)

El bloque `location /api/` redirige al contenedor `backend_app` usando el nombre del servicio como hostname Docker. Esto funciona porque ambos contenedores están en la misma red `backend_network`.

---

### Paso 8: Actualizar `docker-compose.yml` con el Frontend

```yaml
version: '3.8'
services:
  postgres_db:
    image: postgres:16-alpine
    container_name: postgres_db
    restart: always
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    ports:
      - "${POSTGRES_PORT}:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - backend_network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5

  pgadmin:
    image: dpage/pgadmin4:8.4
    container_name: pgadmin
    restart: always
    environment:
      PGADMIN_DEFAULT_EMAIL: ${PGADMIN_DEFAULT_EMAIL}
      PGADMIN_DEFAULT_PASSWORD: ${PGADMIN_DEFAULT_PASSWORD}
    ports:
      - "${PGADMIN_PORT}:80"
    volumes:
      - pgadmin_data:/var/lib/pgadmin
    networks:
      - backend_network
    depends_on:
      - postgres_db

  backend_app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: backend_app
    restart: on-failure
    environment:
      DB_SERVER: postgres_db
      DB_PORT: 5432
      DB_NAME: ${POSTGRES_DB}
      DB_USER: ${POSTGRES_USER}
      DB_PASSWORD: ${POSTGRES_PASSWORD}
      SPRING_PROFILES_ACTIVE: ${SPRING_PROFILES_ACTIVE}
    ports:
      - "8081:8081"
    networks:
      - backend_network
    depends_on:
      postgres_db:
        condition: service_healthy

  frontend_app:
    build:
      context: ../frontend-users
      dockerfile: Dockerfile
    container_name: frontend_app
    restart: always
    ports:
      - "3000:80"
    networks:
      - backend_network
    depends_on:
      - backend_app

volumes:
  postgres_data:
  pgadmin_data:

networks:
  backend_network:
    driver: bridge
```

**`docker-compose.yml` en Visual Studio Code:**

![docker-compose.yml con los 4 servicios orquestados](https://raw.githubusercontent.com/carojas1/semana9-tendencias_tecnologicas/assets/screenshots/vsc_dockercompose.png)

---

### Paso 9: Levantar todos los contenedores

```bash
docker compose up --build -d
```

---

## 9. Diagrama de la Solución

```
┌──────────────────────────────────────────────────────────────┐
│                   Navegador / Host                           │
└────────┬─────────────────────────┬──────────────────────────┘
         │ Puerto 3000             │ Puerto 8090
         │                         │
┌────────▼──────────┐   ┌──────────▼───────────────┐
│   frontend_app    │   │         pgadmin           │
│   nginx:alpine    │   │   dpage/pgadmin4:8.4      │
│  React (Vite)     │   │   Administrador visual    │
└────────┬──────────┘   └──────────┬────────────────┘
         │ /api/ proxy_pass         │
         │ Puerto 8081              │
┌────────▼──────────────────────────▼────────────────┐
│                  backend_network (bridge)           │
│  ┌─────────────────────────────────────────────┐   │
│  │            backend_app                      │   │
│  │         Spring Boot Kotlin                  │   │
│  │           Puerto 8081                       │   │
│  └──────────────────┬──────────────────────────┘   │
│                     │ JPA / JDBC                    │
│  ┌──────────────────▼──────────────────────────┐   │
│  │            postgres_db                      │   │
│  │          PostgreSQL 16-alpine               │   │
│  │            Puerto 5432                      │   │
│  └──────────────────┬──────────────────────────┘   │
│                     │                               │
│              Volumen postgres_data                  │
└─────────────────────────────────────────────────────┘
```

---

## 10. Resultados Obtenidos

### Terminal — docker ps (Contenedores activos)

Los cuatro contenedores están corriendo: `frontend_app` en el puerto 3000, `backend_app` en el 8081, `pgadmin` en el 8090 y `postgres_db` en el 5432 con estado `healthy`.

![docker ps mostrando los 4 contenedores activos](https://raw.githubusercontent.com/carojas1/semana9-tendencias_tecnologicas/assets/screenshots/04_docker_ps.png)

---

### Backend API — Respuesta GET /users (localhost:8081/users)

El endpoint del backend retorna el array JSON con todos los usuarios registrados en PostgreSQL.

![Respuesta JSON del endpoint GET /users](https://raw.githubusercontent.com/carojas1/semana9-tendencias_tecnologicas/assets/screenshots/02_api_json.png)

---

### Frontend React — Tabla de Usuarios (localhost:3000)

El frontend sirve la aplicación React en el puerto 3000. La tabla muestra correctamente los usuarios obtenidos desde la API a través del proxy nginx.

![Frontend React con tabla de usuarios funcionando en Docker](https://raw.githubusercontent.com/carojas1/semana9-tendencias_tecnologicas/assets/screenshots/01_frontend_completo.png)

---

## 11. Conclusiones

Con Docker Compose automaticé el despliegue de los cuatro servicios con un solo comando, lo cual simplifica mucho el proceso y evita errores de configuración manual.

La técnica Multi-Stage Build me ayudó a reducir el tamaño de las imágenes. En el frontend, la imagen final con nginx pesa mucho menos que si hubiéramos dejado Node.js completo, porque solo se copian los archivos estáticos compilados.

Configurar nginx como proxy inverso fue la parte más interesante. Permite que el frontend haga peticiones a rutas relativas (`/api/users`) sin necesidad de conocer el puerto o la dirección del backend. Esto es posible porque los contenedores dentro de la misma red Docker se comunican usando el nombre del servicio como hostname.

Usar un archivo `.env` es una buena práctica de seguridad porque evita poner datos sensibles directamente en el código o en el `docker-compose.yml`.

El uso de `depends_on` con `healthcheck` en Docker Compose fue clave para garantizar el orden correcto de arranque: primero PostgreSQL, luego el backend (que espera a que la BD esté lista), y finalmente el frontend.

Esta práctica me ayudó a entender cómo funciona una arquitectura full-stack contenerizada y cómo aplicar buenas prácticas de DevOps en un proyecto real.

---

## 12. Bibliografía

Docker Inc. (2024a). *Multi-stage builds*. Docker Documentation. https://docs.docker.com/build/building/multi-stage/

Docker Inc. (2024b). *Docker Compose overview*. Docker Documentation. https://docs.docker.com/compose/

Flywaydb. (2024). *Flyway by Redgate: Database migrations made easy*. https://flywaydb.org/documentation/

Meta. (2024). *React — The library for web and native user interfaces*. https://react.dev/

Nginx Inc. (2024). *nginx reverse proxy*. https://nginx.org/en/docs/http/ngx_http_proxy_module.html

Pivotal Software. (2024). *Spring Boot reference documentation*. https://docs.spring.io/spring-boot/docs/current/reference/html/

PostgreSQL Global Development Group. (2024). *PostgreSQL 16 documentation*. https://www.postgresql.org/docs/16/

You, E. (2024). *Why Vite*. Vite Documentation. https://vitejs.dev/guide/why.html

pgAdmin Development Team. (2024). *pgAdmin 4 documentation*. https://www.pgadmin.org/docs/pgadmin4/latest/
