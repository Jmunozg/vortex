# vortex-deps

Command de Claude Code que construye el manifiesto de dependencias
`.vortex/dependencies.yaml` de un repositorio: **auto-deriva** lo que puede del
código y **solo pregunta** lo que no se puede inferir. Vortex lee estos
manifiestos y arma el grafo de dependencias entre soluciones.

## Instalación

```
/plugin marketplace add Jmunozg/vortex
/plugin install vortex-deps@vortex
```

Luego, dentro de cualquier repo:

```
/vortex-deps
```

## Qué hace

1. **Auto-deriva (no pregunta):** NuGet interno → arista `nuget`; connection
   strings → `shared-data`; `BaseAddress`/clientes → `http`/`grpc`; NuGet de
   tercero → inventario `third_party` (solo `package` + `version`, directas).
2. **Pregunta solo los gaps:** mensajería (async pub/sub), dependencias resueltas
   en runtime desde BD, y cualquier arista dudosa.
3. **Escribe** el `.vortex/dependencies.yaml`, conservando lo declarado a mano.

---

## Referencia del manifiesto

Un archivo por repo, en la raíz: `.vortex/dependencies.yaml`.

**Regla de oro:** cada repo declara **solo sus dependencias salientes**, y solo a
mano lo que Vortex **no** puede auto-derivar.

### Tipos de arista

| `type` | Cuándo | Auto-derivable |
|--------|--------|----------------|
| `grpc` | Llamada síncrona gRPC | Parcial |
| `http` | Llamada síncrona REST/HTTP | Parcial |
| `soap` | Llamada WCF/SOAP a otro servicio (legacy .NET Framework) | Sí (config WCF) |
| `async-publish` | Publica a cola/topic | No → manifiesto |
| `async-consume` | Consume de cola/topic | No → manifiesto |
| `shared-data` | Comparte DB / Redis / Mongo | Sí |
| `nuget` | Referencia un paquete NuGet interno | Sí |

### `source`

| Valor | Significado |
|-------|-------------|
| `auto` | Derivada por Vortex del código/tooling. No la pongas a mano. |
| `manifest` | Declarada en este archivo. Alta confianza, la mantenés vos. |
| `database-query` | Resuelta leyendo una tabla en runtime (origen configurado en Vortex). |

### `confidence`

`high` (determinístico) · `medium` (heurística) · `low` (regex/patrón, posible falso positivo).

### `third_party`

Inventario de NuGets externos: `package` + `version` (solo directas del `.csproj`).
No es parte del grafo; sirve para reportes ("qué proyectos usan tal paquete y versión").

---

## Ejemplo

```yaml
solution: documental-api
depends_on:
  # Síncrona gRPC a otro servicio
  - target: certificate-service
    type: grpc
    source: manifest
    confidence: high

  # Destino resuelto en runtime, pero lista estable -> manifiesto
  - target: erp-core
    type: http
    source: manifest
    confidence: high
    note: endpoint resuelto en runtime vía config, lista estable

  # Mensajería: no auto-derivable -> manifiesto
  - target: notifier
    type: async-publish
    source: manifest
    topic: facturacion.emitida

  # NuGet interno: normalmente lo pone Vortex solo (source: auto)
  - target: dictionary
    type: nuget
    source: auto
    package: Shared.Dictionary
    version: "8.0.1"

third_party:
  - package: Newtonsoft.Json
    version: "13.0.3"
  - package: Serilog
    version: "3.1.1"
```
