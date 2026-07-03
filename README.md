# vortex

Marketplace de plugins de **Vortex** para Claude Code.

## Plugins

- **vortex-deps** — auto-deriva las dependencias de un repositorio
  (`.csproj`, `appsettings`, `Program.cs`) y arma el `.vortex/dependencies.yaml`;
  pregunta solo lo que no puede inferir. Vortex lee esos manifiestos y construye
  el grafo de dependencias entre soluciones.

## Instalación

```
/plugin marketplace add Jmunozg/vortex
/plugin install vortex-deps@vortex
```

Después, dentro de cualquier repo: `/vortex-deps`.

Detalles y referencia del manifiesto en
[`plugins/vortex-deps/README.md`](plugins/vortex-deps/README.md).

## Licencia

MIT — ver [LICENSE](LICENSE).
