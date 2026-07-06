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

Escaneá el repo y detectá dependencias salientes reales. **Detectá primero el
stack** por los archivos presentes (un repo puede ser .NET, Angular/Node, o ambos)
y aplicá el bloque que corresponda.

## Si es .NET / .NET Framework (`*.csproj` o `packages.config`)

1. **NuGet interno (arista del grafo):** paquetes internos de la empresa, sea en
   `<PackageReference>` de los `*.csproj` **o** en `packages.config` (.NET Framework).
   → `type: nuget`, `source: auto`, `confidence: high`, con `package` y `version`.
   Van en `depends_on`.

2. **NuGet de tercero (inventario, NO arista):** el resto de paquetes (de `.csproj`
   o `packages.config`, ej. Newtonsoft, Serilog, EF Core). NO son aristas del grafo.
   Guardá cada uno con `package` y `version` en la sección `third_party`.
   Solo directas (no resuelvas transitivas).

3. **shared-data:** leé connection strings en `appsettings*.json`, `web.config`,
   `app.config` y `docker-compose*.yml`. Cada DB/Redis/Mongo distinta es una arista.
   → `type: shared-data`, `source: auto`, `confidence: high`, con `store`.

4. **http / grpc / soap (síncrono):** buscá `BaseAddress`, clientes Refit/`HttpClient`
   y clientes gRPC en `Program.cs` / `Startup.cs` / `Global.asax` / `appsettings*.json`
   → `type: http` o `grpc`. En .NET Framework, los endpoints WCF en
   `<system.serviceModel><client>` de `web.config` / `app.config` → `type: soap`.
   `source: auto`; `confidence: medium` si el destino se dedujo por heurística,
   `low` si vino de un patrón dudoso.

## Si es Angular / Ionic / Node (`package.json`)

Ionic usa el mismo `package.json` y `environment*.ts` que Angular; si ves
`ionic.config.json` o `capacitor.config.*`, es Ionic pero aplica igual este bloque.

5. **npm interno (arista del grafo):** en `package.json`, las `dependencies` que
   sean librerías internas de la empresa (ver criterio abajo).
   → `type: nuget`, `source: auto`, `confidence: high`, con `package` y `version`.
   Van en `depends_on`. (Reusamos el tipo `nuget` como "paquete interno"; si Vortex
   distingue ecosistemas, usá el campo para marcar que es npm.)

6. **npm de tercero (inventario, NO arista):** el resto de `dependencies` y
   `devDependencies`. Guardá cada una con `package` y `version` en `third_party`.
   Solo directas del `package.json` (no resuelvas transitivas).

7. **http (síncrono):** buscá URLs base de APIs en `src/environments/environment*.ts`
   y en `proxy.conf.json`. Cada backend distinto es una arista.
   → `type: http`, `source: auto`, `confidence: medium`.

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

- **`solution:` es obligatorio y va primero.** Derívalo como slug normalizado del
  nombre del repo (minúsculas, guiones; ej. `Andina_Backend_Dotnet` →
  `andina-backend-dotnet`) y **confirmalo con el usuario** antes de escribir, porque
  es lo que otros repos usan en `target:` para conectar las aristas. Nunca lo omitas.
- Reglas del contrato: solo dependencias **salientes**; `source: auto` para lo
  derivado, `source: manifest` para lo que confirmó el usuario.
- Si ya existe `.vortex/dependencies.yaml`, **conservá las entradas `manifest`**
  existentes y actualizá solo las `auto`. Nunca borres lo manual sin preguntar.
- Escribí YAML válido y ordenado (primero `nuget`/`shared-data`, luego síncronas,
  luego mensajería).
- Escribí también la sección `third_party` (lista de `package` + `version` de los
  paquetes externos, NuGet o npm). Va aparte de `depends_on`, no es parte del grafo.
- Al final, mostrá un resumen: cuántas aristas por tipo, cuántos paquetes de tercero,
  y cuáles quedaron marcadas para revisar.

Criterio "paquete interno" (para separar arista vs inventario): en npm, se consideran
internas las librerías con scope de la empresa (ej. `@miorg/...`) o publicadas en el
feed privado; el resto es tercero. Ajustá este criterio a la convención real del equipo.

Tipos válidos: `grpc`, `http`, `soap`, `async-publish`, `async-consume`, `shared-data`, `nuget`.
Valores de `source`: `auto`, `manifest`, `database-query`.
Valores de `confidence`: `high`, `medium`, `low`.
