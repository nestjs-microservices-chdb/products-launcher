# AGENTS.md

Repo raíz tipo **launcher**: solo contiene `docker-compose.yml`, `.env.template` y `.gitmodules`. Los 4 directorios de servicios son **git submodules** (cada uno es su propio repo git, con su propio `package.json` y `node_modules`).

## Submodules (leer antes de commitar)

- Tras un clone: `git submodule update --init --recursive` — sin esto los directorios de servicios están vacíos.
- Los commits hechos dentro de `client-gateway/`, `product-ms/`, etc. van al repo del submodule, **no** a este.
- Para actualizar referencias: **primero** commit+push en el submodule, **después** commit del puntero actualizado aquí. Al revés se pierden referencias y hay conflictos.

## Arquitectura

NestJS 12 con transporte **NATS** en todos los servicios (ver `<svc>/src/main.ts` y `<svc>/src/transports/nast.module.ts` — el archivo se llama `nast` con typo existente; el token de inyección es `NATS_SERVICE` en client-gateway y `NAST_SERVICE` en order-ms/payment-ms).

- `client-gateway` — único punto de entrada HTTP (puerto 3000, prefijo global `api`). REST `/api/products` y `/api/orders` → `ClientProxy.send` por NATS.
- `product-ms` — microservicio NATS puro (sin HTTP: nada que testear con curl/supertest). SQLite + Prisma. Patrones `{ cmd: 'create_product' | 'find_all_products' | 'find_one_product' | 'update_product' | 'remove_product' | 'validate_products' }`.
- `order-ms` — microservicio NATS puro. Postgres + Prisma. Patrones `createOrder`, `findAllOrders`, `findOneOrder`, `changeOrderStatus`. `createOrder` y `findOneOrder` llaman a product-ms (`validate_products`): sin product-ms arriba fallan.
- `payment-ms` — híbrido HTTP + NATS: patrón `create.payment.session` + endpoints HTTP `payments/success`, `payments/cancel`, `payments/webhook` (webhook de Stripe, requiere `rawBody`). Emite evento `payment.succeeded` (aún sin consumidor).
- Errores entre servicios: `RpcException`; el gateway los traduce con `RpcCustomExceptionFilter`.

## Correr todo (docker)

1. Copiar `.env.template` → `.env` en la raíz. Requiere claves de Stripe de prueba (`STRIPE_SECRET`, `STRIPE_WEBHOOK_SECRET`) — payment-ms muere al arrancar sin ellas (zod valida al importar).
2. `docker compose up` — levanta nats-server, los 4 servicios y orders-db (postgres 16.2).

- Cada servicio corre `npm run start:dev` con `./<svc>/src` montado como volumen → hot reload con cambios del host.
- En `order-ms` y `product-ms`, `start:dev` primero ejecuta `prisma migrate dev && prisma generate` (script `docker:start`) — las migraciones se aplican al arrancar el contenedor.
- NATS: el compose raíz solo publica `8222` (monitoring) al host; el puerto cliente `4222` **no** está expuesto → un proceso local no puede apuntar a `localhost:4222` contra este compose.
- Puertos host: gateway `CLIENT_GATEWAY_PORT` (3000), products 3001, orders 3002, payments 3003, postgres 5432.

## Desarrollo local por servicio

Cada subdirectorio es un proyecto autónomo con su propio `.env.template` (apunta `NATS_SERVERS` a `localhost`). Para NATS local: `docker run -d --name nats-server -p 4222:4222 -p 8222:8222 nats`.

Variables requeridas (validadas con zod en `<svc>/src/config/envs.ts`; la app muere al arrancar si falta alguna). `NATS_SERVERS` es lista separada por comas:

- client-gateway: `PORT`, `NATS_SERVERS`
- product-ms / order-ms: + `DATABASE_URL`
- payment-ms: + `STRIPE_SECRET`, `STRIPE_WEBHOOK_SECRET`, `STRIPE_SUCCESS_URL`, `STRIPE_CANCEL_URL` (deben ser URLs válidas)

`order-ms/docker-compose.yml` levanta solo su Postgres (mismo contenedor y volumen que el compose raíz: `orders_database`, user `postgres`, pass `123456`, db `ordersdb`).

## Prisma 7 (product-ms y order-ms)

- Config del CLI: `prisma7.config.ts` en la raíz del servicio — autodetectado, sin `--config`. `DATABASE_URL` vive ahí vía dotenv, no en `schema.prisma`.
- Cliente generado en `src/generated/prisma` — código generado, no editar a mano. Importarlo como `./generated/prisma/client.js`.
- Tras cambiar `prisma/schema.prisma`: `npx prisma migrate dev --name <nombre>` **y luego** `npx prisma generate` (en v7 `migrate dev` NO regenera el cliente).
- Driver adapters obligatorios en v7: `@prisma/adapter-pg` (order-ms, `src/prisma.service.ts`) y `@prisma/adapter-better-sqlite3` (product-ms, `src/primsa.service.ts` — typo existente en el nombre del archivo).
- product-ms: `dev.db` en la raíz del servicio y seed manual en `data/products.sql`. En docker usa `DATABASE_URL=file:/usr/src/app/dev.db` **sin volumen que la persista**: la db vive dentro del contenedor.

## Comandos (idénticos en los 4 servicios; correr dentro del subdirectorio)

- `npm run start:dev` — watch (en order-ms/product-ms corre migraciones antes).
- `npm run lint` — **oxlint** type-aware (`.oxlintrc.json`), no ESLint.
- `npm run build` — `nest build`. `npm run format` — Prettier (`singleQuote`, `trailingComma: "all"`).
- `npm test` — **falla hoy en todos**: "No test files found" (no hay specs unitarios). No es regresión.
- `npm run test:e2e` — **falla hoy en todos**: `test/app.e2e-spec.ts` es el scaffold HTTP de Nest y product-ms/order-ms no son servidores HTTP. No es regresión.
- Un solo test: `npx vitest run <archivo>` o `npx vitest run -t "<nombre>"`.

## Convenciones compartidas

- ESM: `"type": "module"` + tsconfig `module: nodenext` → todo import relativo lleva sufijo `.js` (ej. `./app.module.js`).
- `ValidationPipe` global con `whitelist` + `forbidNonWhitelisted`: propiedades extra en el payload se rechazan — agregar el decorador al DTO.
- Barrel `index.ts` por carpeta (`src/config`, `src/orders/dto`, etc.).

## Otros AGENTS.md

- `order-ms/AGENTS.md` (vive en el submodule order-ms): el detalle de Prisma 7 y comandos sigue vigente, pero su sección de transporte está **desactualizada** — dice TCP; desde la migración es NATS (ver `order-ms/src/main.ts`).
