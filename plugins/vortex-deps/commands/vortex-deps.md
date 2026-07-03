---
description: Genera o actualiza .vortex/dependencies.yaml para este repo
argument-hint: "[opcional: nota o alcance]"
allowed-tools: Read, Grep, Glob, Write, Edit
---

Sos un asistente que construye el manifiesto de dependencias `.vortex/dependencies.yaml`
para ESTE repositorio, siguiendo el contrato de Vortex. Contexto extra del usuario: $ARGUMENTS

Trabajás en dos fases: **primero auto-derivás lo que puedas del código**, y **solo
después preguntás por lo que no se puede inferir**. No preguntes cosas que ya están
en el repo.

# Fase 1 — Auto-derivar (no preguntar)

Escaneá el repo y detectá dependencias salientes reales:

1. **NuGet interno (arista del grafo):** en todos los `*.csproj`, los
   `<PackageReference>` de paquetes internos de la empresa.
   → `type: nuget`, `source: auto`, `confidence: high`, con `package` y `version`.
   Van en `depends_on`.

2. **NuGet de tercero (inventario, NO arista):** el resto de `<PackageReference>`
   (paquetes externos, ej. Newtonsoft, Serilog, EF Core). NO son aristas del grafo.
   Guardá cada uno con `package` y `version` en la sección `third_party`.
   Solo directas del `.csproj` (no resuelvas transitivas).

3. **shared-data:** leé connection strings en `appsettings*.json`, `web.config` y
   `docker-compose*.yml`. Cada DB/Redis/Mongo distinta es una arista.
   → `type: shared-data`, `source: auto`, `confidence: high`, con `store`.

4. **http / grpc (síncrono):** buscá `BaseAddress`, clientes Refit/`HttpClient`,
   y clientes gRPC en `Program.cs` / `Startup.cs` / `appsettings*.json`.
   → `type: http` o `grpc`, `source: auto`. Poné `confidence: medium` si el
   destino se dedujo por heurística, `low` si vino de un patrón dudoso.

Para cada arista, resolvé el `target` a un slug de servicio consistente. Si no
podés mapear el destino con certeza a un servicio conocido, NO la inventes:
marcala como pendiente de confirmar en la Fase 2.

# Fase 2 — Preguntar solo los gaps

Después de mostrar lo auto-derivado, preguntá de forma concreta y una cosa a la vez:

1. **Mensajería (no se auto-deriva):** ¿este servicio publica o consume de alguna
   cola/topic (RabbitMQ/MassTransit)? Por cada una, pedí `topic`/`exchange` y si es
   `async-publish` o `async-consume`.

2. **Dependencias resueltas en runtime desde BD:** ¿hay servicios que este repo
   llama según una tabla en runtime (patrón multi-tenant, ej. clientes)? Si la lista
   es estable, se declaran como `http`/`grpc` con `source: manifest` y una `note`.
   Si cambia seguido, avisá que eso va como origen `database-query` en Vortex, no acá.

3. **Librerías compartidas no-NuGet** o cualquier arista que hayas marcado como
   dudosa en la Fase 1: confirmá `target` y `type` con el usuario.

# Fase 3 — Escribir el archivo

- Reglas del contrato: solo dependencias **salientes**; `source: auto` para lo
  derivado, `source: manifest` para lo que confirmó el usuario.
- Si ya existe `.vortex/dependencies.yaml`, **conservá las entradas `manifest`**
  existentes y actualizá solo las `auto`. Nunca borres lo manual sin preguntar.
- Escribí YAML válido y ordenado (primero `nuget`/`shared-data`, luego síncronas,
  luego mensajería).
- Escribí también la sección `third_party` (lista de `package` + `version` de los
  NuGets externos). Va aparte de `depends_on`, no es parte del grafo.
- Al final, mostrá un resumen: cuántas aristas por tipo, cuántos NuGets de tercero,
  y cuáles quedaron marcadas para revisar.

Tipos válidos: `grpc`, `http`, `async-publish`, `async-consume`, `shared-data`, `nuget`.
Valores de `source`: `auto`, `manifest`, `database-query`.
Valores de `confidence`: `high`, `medium`, `low`.
