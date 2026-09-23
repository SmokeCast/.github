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

## Estructura del proyecto

La carpeta de trabajo agrupa repositorios independientes. Cada microservicio
puede versionarse, construirse y desplegarse sin depender del código fuente de
los demás:

```text
SmokeCast/
├── ms1-fire-catalog/          # Java + Spring Boot + MySQL
├── ms2-urban-exposure/        # Node.js + Drizzle + PostgreSQL
├── ms3-atmosphere-feed/       # Python + FastAPI + MongoDB
├── ms4-smoke-brain/           # Node.js, composición de servicios
├── ms5-analytics-gateway/     # Python + Athena
├── frontend-web/              # React + Vite
├── data-ingestion/            # tres workers pull hacia S3
├── bulk-samples/              # ejemplos para endpoints bulk
└── iac_containers_and_scripting/
    ├── containers-and-seeds/  # Compose de bases, seeds e imágenes
    ├── iac_cloudformation/    # plantillas de AWS
    └── image-publishing/      # build y push a Docker Hub
```

Cada repositorio contiene su propio README, Dockerfile cuando corresponde,
archivo `env.example`, dependencias y documentación Swagger.

## Flujo funcional

1. MS1 registra eventos de fuego y sus detecciones.
2. MS2 encuentra ciudades y sitios sensibles cercanos a las coordenadas del
   incendio.
3. MS3 obtiene o almacena observaciones meteorológicas y de calidad del aire.
4. MS4 combina fuego, exposición y atmósfera para calcular el riesgo de humo.
5. El frontend permite explorar incendios, ciudades, atmósfera y evaluaciones.
6. Los workers de `data-ingestion` extraen el 100% de los registros y los
   depositan en S3 para Glue y Athena.
7. MS5 ejecuta consultas analíticas sobre las tablas y vistas del catálogo.

MS4 no guarda datos propios: si uno de sus servicios dependientes no está
disponible, responde con un error de dependencia para evitar mostrar una
evaluación incompleta.

## Variables de entorno

No hay un único archivo de configuración global. Se usa un `.env` por
proceso o por Compose:

- Los servicios definen su conexión a la base, puerto y URLs de dependencias.
- El Compose de bases define usuarios, contraseñas, nombres de base y puertos.
- El Compose de microservicios define las imágenes, URLs internas y etiquetas.
- La ingesta define credenciales AWS, bucket, prefijos S3 y conexión de lectura.
- MS5 define región, base de datos Athena, workgroup y ubicación de resultados.

Copia siempre el archivo de ejemplo correspondiente:

```bash
cp env.example .env
```

Los valores secretos deben permanecer solamente en la máquina donde se ejecuta
el componente. Para AWS se recomienda usar un perfil o un rol IAM en lugar de
escribir claves permanentes en un archivo.

## Verificación rápida

Después de levantar los servicios se pueden comprobar sus health checks:

```bash
curl http://localhost:8081/actuator/health
curl http://localhost:8082/health
curl http://localhost:8083/health
curl http://localhost:8084/health
curl http://localhost:8085/health
```

Si un servicio devuelve `503`, revisar primero que su base de datos esté
levantada, que el nombre del host coincida con la red Docker y que las
credenciales del `.env` correspondan al Compose de bases.

Para validar un endpoint bulk se puede usar un archivo de
`bulk-samples`:

```bash
curl -X POST http://localhost:8081/api/v1/fires/bulk \
  -H 'Content-Type: application/json' \
  --data-binary @bulk-samples/ms1-fire-events-2000.json
```

Los comandos concretos para MS2 y MS3 están en
`bulk-samples/upload-commands.txt`.

## Criterios cubiertos

El diseño cubre los requisitos funcionales del proyecto:

- Cinco APIs REST documentadas con Swagger UI.
- Tres lenguajes y tres motores de datos: MySQL, PostgreSQL y MongoDB.
- Dos microservicios SQL con tablas relacionadas.
- Un microservicio sin base propia que consume otros tres.
- Un microservicio analítico preparado para Athena, cuatro consultas y dos
  vistas.
- Ingesta pull completa desde las tres fuentes hacia S3.
- Seeder masivo para datos ficticios y endpoints bulk para demostración.
- Plantillas CloudFormation para las máquinas, ALB, API Gateway, VPC Link,
  Glue y los recursos relacionados.

Las decisiones de red, subredes, permisos IAM, balanceo y exposición HTTPS se
terminan de parametrizar durante el despliegue en AWS; el código local sigue
siendo ejecutable sin esos recursos.
