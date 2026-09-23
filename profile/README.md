# SmokeCast

Plataforma de demostración para consultar incendios, ciudades expuestas,
condiciones atmosféricas y riesgo de humo en Latinoamérica.

## Componentes

| Servicio | Tecnología | Puerto | Base de datos |
|---|---|---:|---|
| MS1 Fire Catalog | Java 21 / Spring Boot | 8081 | MySQL 8 |
| MS2 Urban Exposure | Node.js 22+ / Express / Drizzle | 8082 | PostgreSQL 16 |
| MS3 Atmosphere Feed | Python 3.11+ / FastAPI | 8083 | MongoDB 7 |
| MS4 Smoke Brain | Node.js 22+ / Express | 8084 | Consume MS1, MS2 y MS3 |
| MS5 Analytics Gateway | Python 3.11+ / FastAPI | 8085 | Amazon Athena |

El frontend está en `frontend-web`. La ingesta está en `data-ingestion` y el
seeder en `iac_containers_and_scripting/containers-and-seeds/mv_databases/seeds`.

## Ejecución local

1. Copia cada `env.example` a `.env` y completa las credenciales.
2. Levanta las bases:

   ```bash
   cd iac_containers_and_scripting/containers-and-seeds/mv_databases
   cp env.example .env
   docker compose up -d
   ```

3. Ejecuta el seeder si necesitas datos de prueba. Consulta su
   [README](iac_containers_and_scripting/containers-and-seeds/mv_databases/seeds/README.md).
4. Arranca los microservicios desde sus carpetas:

   ```bash
   # MS1
   ./mvnw spring-boot:run

   # MS2, MS4 y frontend
   npm ci && npm start

   # MS3 y MS5
   python -m venv .venv
   source .venv/bin/activate
   python -m pip install -r requirements.txt
   python main.py
   ```

   MS4 requiere que MS1, MS2 y MS3 estén disponibles. MS5 funciona localmente
   con `ATHENA_ENABLED=false` y habilita las consultas al configurar AWS.

5. Ejecuta el frontend desde `frontend-web` con `npm run dev`.

## Documentación de APIs

Cada microservicio publica Swagger UI en `/docs` y OpenAPI en `/openapi.json`:

```text
http://localhost:8081/docs   MS1
http://localhost:8082/docs   MS2
http://localhost:8083/docs   MS3
http://localhost:8084/docs   MS4
http://localhost:8085/docs   MS5
```

Todos los endpoints usan el prefijo `/api/v1`. Los endpoints bulk están
documentados en Swagger y en `bulk-samples/upload-commands.txt`.

## Docker e ingesta

- Compose de bases: `containers-and-seeds/mv_databases`.
- Compose de microservicios: `containers-and-seeds/mv_microservicios`.
- Compose de ingesta: `containers-and-seeds/mv_ingesta`.
- Las imágenes se publican con los scripts de
  `iac_containers_and_scripting/image-publishing`.

La ingesta realiza un pull completo de las tablas de MS1/MS2 y de la colección
de MS3, genera CSV o JSON Lines y los carga en S3. Es un proceso puntual:
ejecuta los tres servicios con `docker compose run --rm` cuando quieras repetirla.

## Correspondencia con la rúbrica

- Cinco microservicios; MS1, MS2 y MS3 usan tres lenguajes y tres bases distintas.
- MS1 y MS2 tienen tablas SQL relacionadas; MS3 usa documentos MongoDB.
- MS4 no tiene base propia y consume los otros servicios.
- MS5 contiene cuatro consultas Athena y dos vistas predefinidas.
- El seeder carga más de 20.000 registros por base en una ejecución.
- Swagger UI está disponible para las cinco APIs.

La infraestructura AWS, el balanceador, API Gateway, Glue, S3 y Athena se
despliegan aparte con las plantillas de `iac_cloudformation`.
