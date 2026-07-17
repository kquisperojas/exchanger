# Sistema de Tipo de Cambio — API Experiencia + API Soporte

Sistema compuesto por dos microservicios en **Java 17 + Spring Boot 4 (WebFlux, reactivo)**, con persistencia en **H2** (R2DBC), seguridad **JWT**, manejo centralizado de **excepciones** y documentación **Swagger/OpenAPI**.

- **API Experiencia** (`:8080`): recibe las solicitudes del cliente, valida JWT, consulta [GoRest](https://gorest.co.in/) para verificar el usuario, obtiene la tasa y registra el historial a través de la API Soporte.
- **API Soporte** (`:8081`): única capa que accede a la base de datos. Expone el CRUD de tipo de cambio y el registro de historial/auditoría. Protegida con API Key interna.

---

## 1. Requisitos previos

| Herramienta | Versión mínima |
|---|---|
| JDK | 17 |
| Maven | 3.9+ |
| Postman o SoapUI | cualquier versión reciente |
| Conexión a internet | necesaria (consumo de `gorest.co.in`) |

Verificar instalación:
```bash
java -version
mvn -version
```

---

## 2. Estructura del proyecto

```
tipo-cambio-parent/
├── api-soporte/        # Microservicio de persistencia (BD H2)
└── api-experiencia/    # Microservicio orientado al cliente (JWT + GoRest)
```

Cada carpeta es un proyecto Maven independiente con su propio `pom.xml`.

---

## 3. Configuración

Toda la configuración ya viene lista en `application.yml` de cada servicio (no requiere pasos manuales adicionales). Los valores relevantes son:

### `api-soporte/src/main/resources/application.yml`
```yaml
server:
  port: 8081
spring:
  r2dbc:
    url: r2dbc:h2:mem:///soportedb;DB_CLOSE_DELAY=-1
security:
  internal:
    api-key: clave-interna-soporte-123
```

### `api-experiencia/src/main/resources/application.yml`
```yaml
server:
  port: 8080
spring:
  r2dbc:
    url: r2dbc:h2:mem:///experienciadb;DB_CLOSE_DELAY=-1
jwt:
  secret: mi_secreto_super_seguro_de_al_menos_32_caracteres
  expiration-ms: 3600000
soporte:
  base-url: http://localhost:8081
  api-key: clave-interna-soporte-123
gorest:
  base-url: https://gorest.co.in/public/v2
```

> ⚠️ **Importante**: el valor de `soporte.api-key` en API Experiencia debe coincidir exactamente con `security.internal.api-key` en API Soporte.

Las bases de datos H2 son **en memoria**: se crean automáticamente al levantar cada servicio (script `schema.sql` incluido) y se pierden al detenerlo. No requieren instalación de motor de BD externo.

---

## 4. Pasos para ejecutar el proyecto

### Paso 1 — Levantar API Soporte
```bash
cd api-soporte
mvn clean spring-boot:run
```
Esperar el mensaje de arranque en el puerto **8081**.

### Paso 2 — Levantar API Experiencia (en otra terminal)
```bash
cd api-experiencia
mvn clean spring-boot:run
```
Esperar el mensaje de arranque en el puerto **8080**.

> Ambos servicios deben quedar corriendo en simultáneo. Levantar siempre primero **API Soporte**, ya que **API Experiencia** depende de ella.

### Paso 3 — Verificar que están arriba
```bash
curl http://localhost:8081/soporte/tipo-cambio -H "x-api-key: clave-interna-soporte-123"
curl http://localhost:8080/api/auth/login
```

---

## 5. Documentación Swagger

| Servicio | URL Swagger UI | URL OpenAPI JSON |
|---|---|---|
| API Soporte | http://localhost:8081/swagger-ui.html | http://localhost:8081/v3/api-docs |
| API Experiencia | http://localhost:8080/swagger-ui.html | http://localhost:8080/v3/api-docs |

Desde Swagger UI de **API Experiencia** puedes usar el botón **Authorize** para pegar el token JWT (`Bearer <token>`) y probar los endpoints protegidos directamente desde el navegador.

---

## 6. Cómo probar el flujo completo (Postman / SoapUI / curl)

### Paso 1 — Obtener un ID de usuario válido de GoRest
```bash
curl https://gorest.co.in/public/v2/users
```
Copiar cualquier `id` de la respuesta (ejemplo: `7654321`).

### Paso 2 — Login para obtener el token JWT

**Usuarios demo disponibles:**

| username | password |
|---|---|
| admin | 1234 |
| analista | abcd |

```bash
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"1234"}'
```
Respuesta:
```json
{ "token": "eyJhbGciOiJIUzI1NiJ9..." }
```
Copiar el valor de `token`.

### Paso 3 — Ejecutar el tipo de cambio
```bash
curl -X POST http://localhost:8080/api/exchange \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <TOKEN>" \
  -d '{
        "userId": <ID_GOREST>,
        "monto": 100,
        "monedaOrigen": "USD",
        "monedaDestino": "PEN"
      }'
```
Respuesta esperada:
```json
{
  "mensaje": "Tipo de cambio realizado con exito",
  "usuario": "Nombre Obtenido de GoRest",
  "montoInicial": 100.0,
  "monedaOrigen": "USD",
  "monedaDestino": "PEN",
  "tasa": 3.75,
  "montoFinal": 375.0,
  "idHistorial": 1
}
```

### Paso 4 — Verificar historial y auditoría en API Soporte
```bash
curl http://localhost:8081/soporte/historial -H "x-api-key: clave-interna-soporte-123"
curl "http://localhost:8081/soporte/tipo-cambio?origen=USD&destino=PEN" -H "x-api-key: clave-interna-soporte-123"
```

---

## 7. Endpoints disponibles

### API Experiencia (`http://localhost:8080`)

| Método | Endpoint | Auth | Descripción |
|---|---|---|---|
| POST | `/api/auth/login` | No | Login, devuelve JWT |
| POST | `/api/exchange` | JWT Bearer | Realiza el tipo de cambio |

### API Soporte (`http://localhost:8081`)

| Método | Endpoint | Auth | Descripción |
|---|---|---|---|
| POST | `/soporte/tipo-cambio` | x-api-key | Registrar nueva tasa |
| PUT | `/soporte/tipo-cambio/{id}` | x-api-key | Actualizar tasa existente |
| GET | `/soporte/tipo-cambio?origen=&destino=` | x-api-key | Buscar tasa por par de monedas (sin parámetros: lista todas) |
| POST | `/soporte/historial` | x-api-key | Registrar historial de un cambio realizado |
| GET | `/soporte/historial` | x-api-key | Listar historial completo |

**Tasas precargadas por defecto:** USD→PEN (3.75), PEN→USD (0.2667), EUR→PEN (4.05).

---

## 8. Colección para Postman / SoapUI

Sugerencia de organización de la colección:

1. **Auth** → `Login` (guarda el token en variable de entorno `{{token}}`)
2. **Exchange** → `Realizar tipo de cambio` (usa `Authorization: Bearer {{token}}`)
3. **Soporte - Tipo de Cambio** → `Crear`, `Actualizar`, `Buscar`
4. **Soporte - Historial** → `Registrar`, `Listar`

Variables de entorno recomendadas:
```
base_url_experiencia = http://localhost:8080
base_url_soporte     = http://localhost:8081
api_key_soporte       = clave-interna-soporte-123
token                 = (se llena tras el login)
```

---

## 9. Manejo de errores esperado

| Escenario | Código HTTP | Respuesta |
|---|---|---|
| Sin token / token inválido | 401 | `{"error": "Token no proporcionado"}` / `"Token invalido o expirado"` |
| Usuario no existe en GoRest | 404 | `{"mensaje": "Usuario con id X no existe en GoRest"}` |
| Tipo de cambio no configurado | 404 | `{"mensaje": "Tipo de cambio no configurado para X -> Y"}` |
| Campos faltantes / inválidos | 400 | `{"mensaje": "campo: detalle del error"}` |
| API Key inválida (API Soporte) | 401 | `{"error": "API key invalida o no proporcionada"}` |
| Error interno | 500 | `{"mensaje": "Error interno: ..."}` |

Todas las respuestas de error siguen el mismo formato estándar (`timestamp`, `status`, `error`, `mensaje`), generado por el `GlobalExceptionHandler` de cada servicio.

---

## 10. Notas finales

- Las bases H2 son en memoria: al reiniciar cualquiera de los servicios, los datos (historial, auditoría, tasas creadas manualmente) se reinician, excepto las tasas precargadas por `schema.sql`.
- `gorest.co.in` es un servicio público de pruebas: los usuarios existentes pueden variar; siempre validar con un `GET /public/v2/users` antes de probar.
- Para producción: reemplazar H2 por una base persistente (PostgreSQL/MySQL) cambiando únicamente la configuración `r2dbc.url` y el driver en el `pom.xml`; el resto del código no requiere cambios.
