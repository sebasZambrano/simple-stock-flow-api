# simple-stock-flow-api


## Cómo se clona

El sistema vive en repositorios separados y **el compose construye desde las carpetas hermanas**,
así que la disposición no es cosmética: hay que clonarlos en el mismo directorio y con su nombre.

```bash
mkdir simple-stock-flow && cd simple-stock-flow
git clone https://github.com/code-dev-projects/simple-stock-flow-infra.git
git clone https://github.com/code-dev-projects/simple-stock-flow-api.git
git clone https://github.com/code-dev-projects/simple-stock-flow-portal.git
```

```
simple-stock-flow/
├── simple-stock-flow-infra/     compose, .env.example y verify.sh
├── simple-stock-flow-api/       backend .NET 8
└── simple-stock-flow-portal/    front Angular 20
```

`simple-stock-flow-docs` no hace falta para levantar el sistema: guarda el material de trabajo.

## 1. Qué es esto

El backend REST del sistema de Productos y Ventas: catálogo, registro de ventas y reporte por
rango de fechas, sobre .NET 8 + PostgreSQL con arquitectura hexagonal. **Es dueño del esquema
de la base**: las migraciones viven aquí y se aplican al arrancar.

De lo que **no** se ocupa: no sirve la interfaz de usuario (eso es `simple-stock-flow-portal`), no
define el `docker compose` ni el `.env` del sistema (eso es `simple-stock-flow-infra`) y no crea la
base de datos —eso lo hace el contenedor de Postgres; este servicio solo crea el esquema
dentro de ella—.

> **Antes de seguir, lo que un evaluador necesita saber:** el sistema **no se puede usar de
> punta a punta hoy**. `POST /api/auth/login` responde **500**, así que no hay forma de obtener
> un token y todo lo demás responde 401. El detalle honesto está en la §5.

---

## 2. Cómo se levanta

Hay dos caminos. El primero es el del sistema completo; el segundo es para trabajar solo sobre
la API.

### A) El sistema completo (API + Postgres + portal)

Necesita **Docker**. El `docker compose` no está en este repositorio, está en
`simple-stock-flow-infra`, que es un repo hermano dentro del mismo workspace:

```bash
cd ../simple-stock-flow-infra
cp .env.example .env          # rellena POSTGRES_PASSWORD, JWT_SIGNING_KEY y ADMIN_PASSWORD
docker compose up -d
docker compose ps             # los tres contenedores deben quedar (healthy)
```

Queda publicado un único puerto al host: **`http://localhost:8080`** (valor de `PORTAL_PORT`).
Ahí responde el portal, y su nginx hace de proxy hacia la API bajo `/api/` y `/media/`. El
contenedor de la API **no publica puerto propio**, y `/health` no está proxiado: para verlo hay
que entrar por dentro de la red de Docker.

```bash
docker compose exec db wget -q -O - http://service:8080/health
# {"status":"ok"}
```

### B) Solo este servicio, en local

Necesita el **SDK de .NET** (los proyectos apuntan a `net8.0`; se verificó compilando y
ejecutando con el SDK 10.0.401) y un Postgres accesible. Las dos variables de abajo son
**obligatorias** y no tienen valor por defecto útil:

```bash
export ConnectionStrings__Postgres="Host=localhost;Port=5432;Database=simple_stock_flow;Username=simple_stock_flow;Password=LA_DEL_ENV"
export Jwt__SigningKey="una-clave-de-al-menos-32-caracteres-aqui"
export ASPNETCORE_ENVIRONMENT=Development

dotnet run --project src/bootstrap --urls http://localhost:5080
```

- **`--urls` no es opcional si quieres un puerto predecible.** `dotnet run` lee
  `src/bootstrap/Properties/launchSettings.json`, que fija `https://localhost:62596` y
  `http://localhost:62597`. El argumento `--urls` gana; la variable `ASPNETCORE_URLS` **no**.
- Swagger queda en `http://localhost:5080/swagger`, y solo con `ASPNETCORE_ENVIRONMENT=Development`.
- **Sin `Jwt__SigningKey` de 32 caracteres o más, la API se niega a arrancar** y lanza un
  `InvalidOperationException` que dice exactamente qué falta. Es deliberado: firmar con una
  clave de relleno arranca bien y reparte tokens falsificables. Esto afecta también a
  `dotnet ef` (§3).
- Si apuntas al Postgres del compose desde el host, necesitas el perfil de desarrollo que
  publica el 5432 (§3).

### Variables de entorno que lee el servicio

Doble guion bajo separa niveles de configuración.

| Variable | Qué es | Por defecto |
|---|---|---|
| `ConnectionStrings__Postgres` | Cadena de conexión | `Host=localhost;...;Username=postgres;Password=postgres` (solo sirve en local) |
| `Jwt__SigningKey` | Clave de firma, mínimo 32 caracteres | **ninguno — sin ella no arranca** |
| `Jwt__Issuer` · `Jwt__Audience` · `Jwt__LifetimeMinutes` | Emisor, audiencia y vigencia del token | `simple-stock-flow` · `simple-stock-flow-app` · `60` |
| `Storage__RootPath` | Carpeta donde se escriben los binarios | `/var/lib/simple-stock-flow/media` (en `Development`, `./.media`) |
| `Storage__PublicBaseUrl` | Prefijo con el que se publican | `/media` |
| `Cors__Origins__0` | Origen permitido | `http://localhost:4200` |

`Bootstrap__AdminUsername` y `Bootstrap__AdminPassword` **se le pasan al contenedor desde el
compose, pero el servicio todavía no las lee**: no existe ningún código que las consulte. No
se crea ningún administrador al arrancar.

---

## 3. Dónde están los datos

### La base

| | |
|---|---|
| Motor | PostgreSQL 16 (`postgres:16-alpine`) |
| Base | **`simple_stock_flow`** |
| Usuario | **`simple_stock_flow`** |
| Contraseña | **`POSTGRES_PASSWORD` del archivo `.env` de `simple-stock-flow-infra`.** No se versiona y no tiene valor por defecto: si no lo tienes, no hay dónde leerlo, hay que ponerlo |
| Esquema | **`sales`** — no `public`. `SalesDbContext` hace `HasDefaultSchema("sales")` |
| Puerto | 5432 **dentro** de la red de Docker. **Al host solo se publica con el perfil de desarrollo** (abajo) |
| Volumen | `simple-stock-flow_pgdata`. Los datos sobreviven a `docker compose down`; `down -v` los borra |

Las cinco tablas son `sales.category`, `sales.product`, `sales.sale`, `sales.sale_item` y
`sales.user`, **en singular** por convención del proyecto. `user` no necesita comillas mientras
vaya cualificada por el esquema, que es como se consulta siempre.

### Mirar las filas con tus propios ojos

Sin instalar ningún cliente, con el stack levantado:

```bash
cd ../simple-stock-flow-infra
docker compose exec db psql -U simple_stock_flow -d simple_stock_flow -c "select * from sales.category;"
```

Y para navegar a mano: `docker compose exec db psql -U simple_stock_flow -d simple_stock_flow`, luego
`\dn` (esquemas), `\dt sales.*` (tablas) y `\q`.

### Con un cliente gráfico (DBeaver, pgAdmin, DataGrip)

El 5432 no está publicado en el perfil normal — la base se queda dentro de Docker a propósito.
Para abrirlo hay una superposición de compose:

```bash
cd ../simple-stock-flow-infra
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d
```

Y conectas a `localhost:5432`, base `simple_stock_flow`, usuario `simple_stock_flow`, contraseña la del
`.env`. Recuerda apuntar el cliente al esquema **`sales`**.

### Migraciones

Cuatro, en `src/adapters/outbound/persistence/Migrations/`, y `Program.cs` las aplica al arrancar
(`context.Database.MigrateAsync()`):

| Migración | Qué hace |
|---|---|
| `InitialSchema` | Crea el esquema `sales` con sus cinco tablas y siembra 5 categorías |
| `StockNonNegative` | Restricción de stock no negativo sobre `product` |
| `AccentSeedCategoryNames` | Corrige las tildes de las categorías sembradas |
| `RenameTablesToSingular` | Pasa las cinco tablas a singular, por convención del proyecto. Solo renombra: ninguna columna cambia y ninguna fila se reescribe |

**El historial de migraciones no vive en `sales`**, sino en `public."__EFMigrationsHistory"`
—ahí es donde EF lo pone y donde hay que mirarlo—:

```bash
docker compose exec db psql -U simple_stock_flow -d simple_stock_flow \
  -c 'select "MigrationId" from public."__EFMigrationsHistory";'
```

Para crear una migración nueva hace falta `Jwt__SigningKey` en el entorno: `dotnet ef` construye
el host real de la aplicación, y ese host se niega a arrancar sin la clave.

```bash
Jwt__SigningKey="una-clave-de-al-menos-32-caracteres-aqui" \
dotnet ef migrations add <Nombre> \
  --project src/adapters/outbound/persistence \
  --startup-project src/bootstrap \
  --output-dir Migrations
```

### Las imágenes no están en la base

Los binarios que sube `POST /api/products/{id}/image` **no van a Postgres**. `LocalFileStorage`
los escribe en la carpeta `Storage__RootPath` = `/var/lib/simple-stock-flow/media`, que en el compose
es el volumen `simple-stock-flow_media` (y nunca dentro de la imagen del contenedor). La base solo
guarda la clave del archivo. Se sirven como estáticos bajo `/media/{key}`.

```bash
docker compose exec service ls -la /var/lib/simple-stock-flow/media
```

(En Git Bash sobre Windows, antepón `MSYS_NO_PATHCONV=1` o la ruta absoluta se traduce sola y
el comando falla.)

Hoy esa carpeta está vacía: subir una imagen exige un token de administrador, y no hay forma de
obtener uno (§5).

---

## 4. Cómo se prueba

```bash
dotnet build SimpleStockFlow.sln     # 10 proyectos, 0 errores, 0 warnings
dotnet test SimpleStockFlow.sln      # 37 pruebas
```

`TreatWarningsAsErrors` está activo: cualquier warning rompe la compilación.

| Proyecto | Pruebas | ¿Necesita Docker? |
|---|---|---|
| `tests/Domain.UnitTests` | 12 | **No** |
| `tests/Application.UnitTests` | 12 | **No** — los puertos outbound van en doble (NSubstitute) |
| `tests/Adapters.IntegrationTests` | 13 | **Sí** — levanta un Postgres real con Testcontainers |

```bash
dotnet test tests/Domain.UnitTests          # sin Docker
dotnet test tests/Application.UnitTests     # sin Docker
dotnet test tests/Adapters.IntegrationTests # con Docker corriendo
```

Testcontainers levanta su propio Postgres efímero: **no toca la base del compose** y no hace
falta que el stack esté arriba, solo el demonio de Docker.

---

## 5. Qué falta

Las 37 pruebas están en verde y aun así el sistema no se puede ejercer. Las dos cosas son
ciertas a la vez, y esta sección existe para no esconder la segunda.

**De los 12 métodos de caso de uso, hay 1 implementado y 11 que lanzan
`NotImplementedException`.** Cada uno lleva encima un `TODO` que dice qué hay que hacer.

| Servicio | Métodos | Estado |
|---|---|---|
| `PlaceSaleService` | 1 | **Implementado** |
| `ProductCatalogService` | 6 | Sin implementar |
| `AuthenticationService` | 2 | Sin implementar |
| `GetSaleService` | 2 | Sin implementar |
| `SalesReportService` | 1 | Sin implementar |

Consecuencias que se comprueban con un `curl` contra el sistema levantado:

- **`POST /api/auth/login` → 500.** No hay manera de conseguir un token.
- **Todo lo autenticado → 401**, porque no hay token que presentar. `GET /api/products` sin
  cabecera devuelve 401 correctamente; con token no se ha podido probar nunca.
- **`GET /api/categories` → 404.** El endpoint no existe en el backend y el frontend ya lo
  invoca: es la única brecha del contrato entre los dos repos.
- No hay administrador inicial. Además de que `AuthenticationService` no está implementado, el
  servicio ignora `Bootstrap__AdminUsername`/`AdminPassword` (§2).

Lo que sí está terminado y verificado: el dominio con sus invariantes, el esquema y sus tres
migraciones, la concurrencia optimista, los repositorios EF, el almacenamiento de binarios, la
generación de tokens y el hash de contraseñas, y el arranque completo en contenedor.

---

## Endpoints

`GET /health` y `GET /media/{key}` cuelgan de la raíz; el resto, de `/api`.

| Método | Ruta | Auth |
|---|---|---|
| `POST` | `/api/auth/login` | anónimo |
| `POST` | `/api/auth/register` | `admin` |
| `GET` | `/api/products?search=&categoryId=&page=1&size=20` | autenticado |
| `GET` | `/api/products/{id}` | autenticado |
| `POST` `PUT` `DELETE` | `/api/products[/{id}]` | `admin` |
| `POST` | `/api/products/{id}/image` | `admin` |
| `POST` | `/api/sales` | autenticado |
| `GET` | `/api/sales/{id}` | autenticado |
| `GET` | `/api/sales?from=&to=&page=1&size=20` | autenticado — **`from` y `to` son obligatorios**, sin valor por defecto |
| `GET` | `/api/reports/sales?from=&to=` | autenticado |
| `GET` | `/health` | anónimo |
| `GET` | `/media/{key}` | anónimo (archivo estático) |

---

## La regla del hexágono

Tres reglas que el código hace cumplir, no tres intenciones:

1. **`src/domain/` no referencia nada.** Su `.csproj` no tiene ni un `ItemGroup`, y el
   comentario que hay dentro explica que ese vacío *es* la regla.
2. **`src/application/` solo referencia el dominio.** Habla con el mundo por interfaces que él
   mismo declara, en `Ports/Outbound/`.
3. **`src/bootstrap/Composition/PortBindings.cs` es el único archivo donde un puerto se encuentra
   con su adapter.** Un `new` de infraestructura fuera de ahí significa que el hexágono se rompió.

El código, los nombres y los comentarios están en inglés; esta documentación, en español. Es
deliberado y no hay que "arreglarlo".

En el workspace existen además `simple-stock-flow-docs` (especificaciones, ADR y el enunciado) y
`simple-stock-flow-infra` (compose y `.env`). Nada de lo que hace falta para levantar, consultar o
probar este servicio depende de leer `simple-stock-flow-docs`.

## Licencia

MIT. Copyright (c) 2026 Jesus Ariel Gonzalez Bonilla. El texto completo está en
[`LICENSE`](LICENSE): puede usarse, copiarse, modificarse y distribuirse libremente, con la única
condición de conservar el aviso de copyright.
