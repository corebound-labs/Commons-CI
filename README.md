# Commons-CI

Reusable GitHub Actions workflows compartidos entre proyectos de la
organización [corebound-labs](https://github.com/corebound-labs).
Equivalente al concepto de "shared pipeline templates" de Azure DevOps.

## Workflows disponibles

### `semver-release.yml` — Versionado semántico automático

Calcula la versión semántica del proyecto usando
[GitVersion](https://gitversion.net/) a partir de los mensajes de commit
([Conventional Commits](https://www.conventionalcommits.org/)), y:

1. Crea un tag de Git (`vX.Y.Z`).
2. Crea un GitHub Release con changelog autogenerado.
3. Estampa la versión en los `.csproj` indicados vía `dotnet build /p:Version=...`.

Si la versión calculada coincide con un tag ya existente (p. ej. porque el
último commit fue `docs:` o `chore:` y no debía subir versión), el workflow
no crea tag ni release nuevos.

#### Convención de commits esperada

```
feat: ...          → sube MINOR
fix: ...            → sube PATCH
feat!: ...          → sube MAJOR
BREAKING CHANGE: ... (en el footer del commit) → sube MAJOR
docs:, chore:, style:, refactor:, test:, ci:, build: → no sube versión
```

#### Cómo consumirlo

En el repo consumidor, crear `.github/workflows/release.yml`:

```yaml
name: Release

on:
  push:
    branches: [master]

permissions:
  contents: write

jobs:
  version-and-release:
    uses: corebound-labs/commons-ci/.github/workflows/semver-release.yml@main
    with:
      dotnet-version: '10.0.x'
      csproj-paths: 'EcoTrack/EcoTrack.csproj,EcoTrack.Application/EcoTrack.Application.csproj'
    secrets: inherit
```

El repo consumidor necesita además un archivo `GitVersion.yml` en su raíz
(ver ejemplo en la documentación de [GitVersion](https://gitversion.net/docs/reference/configuration))
que defina cómo interpretar los mensajes de commit para cada tipo de bump.

#### Inputs

| Input            | Requerido | Descripción                                              |
|-------------------|-----------|-----------------------------------------------------------|
| `dotnet-version`  | sí        | Versión del SDK de .NET a instalar (ej. `10.0.x`)         |
| `csproj-paths`    | sí        | Rutas de los `.csproj` a versionar, separadas por coma    |

#### Outputs

| Output         | Descripción                                                |
|-----------------|-------------------------------------------------------------|
| `new-version`   | Versión semántica generada (sin el prefijo `v`)             |
| `tag-created`   | `true`/`false` según si se creó un tag/release nuevo         |

## Alcance

Este repo cubre únicamente reutilización de **pipelines** (YAML de GitHub
Actions). La reutilización de **código** C# compartido (`Commons.CrudOrm`,
`Commons.Infisical`, etc. como paquetes NuGet vía GitHub Packages) es un
alcance distinto, no cubierto aquí.
