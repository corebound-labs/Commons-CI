# Commons-CI

Reusable GitHub Actions workflows compartidos entre proyectos de la
organización [corebound-labs](https://github.com/corebound-labs).
Equivalente al concepto de "shared pipeline templates" de Azure DevOps.

## Workflows disponibles

### `semver-release.yml` — Versionado semántico automático

Calcula la versión del proyecto a partir de los mensajes de commit
([Conventional Commits](https://www.conventionalcommits.org/)) desde el último
tag `v*`, y:

- `feat:` sube el **minor** (`1.309.0` → `1.310.0`); `fix:` sube el **patch**
  (`1.309.0` → `1.309.1`); `feat!:`/`fix!:` (cambio incompatible) suben el
  **major**. Cualquier otro tipo (`docs`, `chore`, `refactor`, `test`, `ci`…)
  **no cambia** `X.Y.Z`: por eso los commits deben ser solo `feat` o `fix` si se
  quiere que la versión avance. Gana el bump más alto entre los commits nuevos
  (incluidos los de ramas fusionadas).
- El sufijo es `-MMddn`: día (`MMdd`, en la zona horaria del input `timezone`,
  por defecto `Europe/Madrid`) seguido del número de build de ese día, sin
  separador. Ej.: `1.310.0-10043` es la tercera versión del 4 de octubre.
  Limitaciones conocidas: desde la build 10 del día el orden NuGet/SemVer deja
  de ser cronológico (`100410` > `10051`), y de enero a septiembre queda un cero
  inicial (`01041`), que SemVer estricto no admite (NuGet sí).
- El núcleo `X.Y.Z` parte del más alto entre los tags `v*` alcanzables; los
  commits "nuevos" son los que ningún tag contiene todavía.
- Un commit que ya tiene tag conserva su versión (un re-run no crea otro tag).

Y después:

1. Crea un tag de Git (`vX.Y.Z-MMddn`).
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

El repo consumidor no necesita ningún archivo de configuración: las reglas
viven en el propio workflow.

#### Inputs

| Input            | Requerido | Descripción                                              |
|-------------------|-----------|-----------------------------------------------------------|
| `dotnet-version`  | sí        | Versión del SDK de .NET a instalar (ej. `10.0.x`)         |
| `csproj-paths`    | sí        | Rutas de los `.csproj` a versionar, separadas por coma    |
| `timezone`        | no        | Zona horaria del `MMdd` del sufijo (default `Europe/Madrid`) |

#### Outputs

| Output         | Descripción                                                |
|-----------------|-------------------------------------------------------------|
| `new-version`   | Versión semántica generada (sin el prefijo `v`)             |
| `tag-created`   | `true`/`false` según si se creó un tag/release nuevo         |

### `pages-deploy.yml` — Deploy a GitHub Pages

Cubre tanto un sitio estático sin build (`docs`, se sube el repo tal cual)
como uno que necesita compilar antes (`cv-martin`, con su propio script de
build). Repo público o privado, de la org o personal: solo hace falta que el
repo consumidor tenga Pages habilitado.

```yaml
# Sin build (ej. docs)
on:
  push:
    branches: [master]

jobs:
  pages:
    uses: corebound-labs/Commons-CI/.github/workflows/pages-deploy.yml@master
    with:
      publish-path: '.' # opcional, por defecto '.'
```

```yaml
# Con build (ej. cv-martin)
on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  pages:
    uses: corebound-labs/Commons-CI/.github/workflows/pages-deploy.yml@master
    with:
      build-command: 'npm run build'
      publish-path: 'dist'
```

| Input             | Requerido | Descripción                                                        |
|-------------------|-----------|------------------------------------------------------------------------|
| `publish-path`    | no        | Carpeta a publicar (def. `.`)                                          |
| `build-command`   | no        | Comando de build antes de publicar (def. vacío = no hay build)         |
| `node-version`    | no        | Node.js, solo si `build-command` no está vacío (def. `22`)             |
| `use-npm-ci`      | no        | `true` usa `npm ci` con cache (requiere lockfile); `false` usa `npm install` (def. `true`) |

### `dotnet-publish-obfuscate.yml` — Publicar + ofuscar JS

Restore, `dotnet publish` y (opcional) ofuscación del JS propio vía
`obfuscate-js`, dejando el resultado en un artifact. Deliberadamente no
incluye el paso de deploy en sí (msdeploy, Azure, lo que sea): cada app
publica en un sitio distinto y eso puede cambiar, así que ese paso vive en el
`deploy.yml` del repo consumidor, en un job separado que descarga el
artifact.

```yaml
jobs:
  build:
    uses: corebound-labs/Commons-CI/.github/workflows/dotnet-publish-obfuscate.yml@master
    with:
      dotnet-version: '10.0.x'
      solution-path: 'EcoTrack.sln'
      csproj-path: 'EcoTrack/EcoTrack.csproj'
      js-path: 'wwwroot/js' # solo el JS propio, no wwwroot/lib ni RCLs

  deploy:
    needs: build
    runs-on: windows-latest # o lo que pida el hosting
    steps:
      - name: Descargar artifact publicado
        uses: actions/download-artifact@v4
        with:
          name: ${{ needs.build.outputs.artifact-name }}
          path: publish
      - name: Deploy
        run: echo "aquí el paso específico del hosting (msdeploy, az webapp, rsync...)"
```

| Input             | Requerido | Descripción                                                        |
|-------------------|-----------|----------------------------------------------------------------------|
| `dotnet-version`  | sí        | Versión del SDK de .NET a instalar (ej. `10.0.x`)                    |
| `solution-path`   | sí        | `.sln` a restaurar (resuelve también proyectos referenciados)        |
| `csproj-path`     | sí        | `.csproj` a publicar                                                 |
| `obfuscate-js`    | no        | Ofuscar el JS propio tras publicar (def. `true`)                     |
| `js-path`         | no*       | Ruta del JS propio a ofuscar, relativa a la carpeta publicada (ej. `wwwroot/js`). Requerido si `obfuscate-js` es `true` |
| `js-exclude`      | no        | Patrones a excluir de la ofuscación, coma (def. `*.min.js`)          |
| `obfuscate-rcl-js`| no        | Ofuscar también el JS de las Razor Class Libraries (ej. `UiMetadata.*`), publicado por ASP.NET Core bajo `wwwroot/_content/<PackageId>/` (def. `true`). Sin esto ese JS queda en claro aunque `obfuscate-js` sea `true` — ver nota abajo |
| `rcl-js-path`     | no        | Carpeta de static web assets de las RCL, relativa a la publicada (def. `wwwroot/_content`) |
| `artifact-name`   | no        | Nombre del artifact publicado (def. `publish`)                       |

> **Nota — JS de RCL:** el JS de EcoTrack (o la app que sea) vive en `wwwroot/js`, pero el JS de cada Razor Class Library referenciada (`UiMetadata.Grid`, `.Elements`, etc.) vive en el `wwwroot/js` de *su propio* proyecto y ASP.NET Core lo copia al publicar bajo `wwwroot/_content/<PackageId>/js/...` — una carpeta distinta a `js-path`, que `obfuscate-js` nunca toca. `obfuscate-rcl-js` (activo por defecto) añade un segundo paso sobre `rcl-js-path` para cubrirlo. A diferencia del JS propio, esta carpeta es opcional: si la app no tiene RCLs, o ninguna trae JS, el paso se omite en vez de fallar el build (`allow-missing`/`allow-empty` en la action).

Output: `artifact-name` (igual al input, para encadenar `needs.build.outputs.artifact-name` en el job de deploy sin repetirlo).

### `dotnet-tests.yml` — Tests .NET

Restore, build en Release y `dotnet test`, publicando resultados con
`dorny/test-reporter`.

```yaml
jobs:
  test:
    uses: corebound-labs/Commons-CI/.github/workflows/dotnet-tests.yml@master
    with:
      dotnet-version: '10.0.x'
      solution-path: 'EcoTrack.sln' # opcional, por defecto '.'
```

| Input             | Requerido | Descripción                                          |
|-------------------|-----------|-------------------------------------------------------|
| `dotnet-version`  | sí        | Versión del SDK de .NET a instalar (ej. `10.0.x`)      |
| `solution-path`   | no        | `.sln`/`.csproj` a restaurar/compilar/testear (def. `.`) |
| `fail-on-empty-test-report` | no | `false` tolera una solución sin ningún proyecto de test (def. `true` = preserva el comportamiento de siempre: sin `.trx` es una falla real) |

### `node-tests.yml` — Tests Node (Vitest/Jest)

Instala Node, dependencias y corre el comando de test indicado en la carpeta
del proyecto.

```yaml
jobs:
  test-js:
    uses: corebound-labs/Commons-CI/.github/workflows/node-tests.yml@master
    with:
      working-directory: 'Commons/UIMetadata/UiMetadata.Grid'
      node-version: '20'   # opcional, def. '20'
      test-command: 'npm test' # opcional, def. 'npm test'
      use-npm-ci: false    # opcional, def. false — true exige package-lock.json commiteado
```

| Input                | Requerido | Descripción                                                        |
|-----------------------|-----------|----------------------------------------------------------------------|
| `working-directory`  | sí        | Carpeta del proyecto Node a testear                                  |
| `node-version`       | no        | Node.js (def. `20`)                                                  |
| `test-command`       | no        | Comando de test (def. `npm test`)                                    |
| `use-npm-ci`         | no        | `true` usa `npm ci` con cache (requiere lockfile); `false` usa `npm install` (def. `false`) |

### `nuget-vulnerability-scan.yml` — Paquetes NuGet vulnerables

Restore + `dotnet list package --vulnerable --include-transitive`, fallando
el job si encuentra algo. Pensado para correr en PR y también en cron
(`schedule`) desde el repo consumidor, para detectar CVEs nuevos sin cambios
de código.

```yaml
on:
  pull_request:
    branches: [develop, master]
  schedule:
    - cron: '0 6 * * 1'
  workflow_dispatch:

permissions:
  contents: read

jobs:
  vulnerable-packages:
    uses: corebound-labs/Commons-CI/.github/workflows/nuget-vulnerability-scan.yml@master
    with:
      dotnet-version: '10.0.x'
      solution-path: 'EcoTrack.sln' # opcional, por defecto '.'
```

| Input             | Requerido | Descripción                                          |
|-------------------|-----------|-------------------------------------------------------|
| `dotnet-version`  | sí        | Versión del SDK de .NET a instalar (ej. `10.0.x`)      |
| `solution-path`   | no        | `.sln`/`.csproj` a restaurar/escanear (def. `.`)       |

### `dotnet-nuget-publish.yml` — Empaquetar y publicar en GitHub Packages

Empaqueta (`dotnet pack`) cada proyecto packable de la solución y lo publica
al feed de GitHub Packages **del propio repo que llama** al workflow, con el
`GITHUB_TOKEN` nativo (no un PAT — eso solo hace falta para *consumir* el
feed desde otro repo, ver más abajo). Pensado para encadenarse después de
`semver-release.yml`, usando la versión que ese job calculó.

```yaml
permissions:
  contents: write
  packages: write

jobs:
  version-and-release:
    uses: corebound-labs/Commons-CI/.github/workflows/semver-release.yml@master
    with:
      dotnet-version: '10.0.x'
      csproj-paths: 'Commons.CrudOrm/Commons.CrudOrm.csproj,...'
    secrets: inherit

  publish:
    needs: version-and-release
    if: needs.version-and-release.outputs.tag-created == 'true'
    uses: corebound-labs/Commons-CI/.github/workflows/dotnet-nuget-publish.yml@master
    with:
      dotnet-version: '10.0.x'
      solution-path: 'Commons.slnx'
      package-version: ${{ needs.version-and-release.outputs.new-version }}
    secrets: inherit
```

| Input             | Requerido | Descripción                                                        |
|-------------------|-----------|------------------------------------------------------------------------|
| `dotnet-version`  | sí        | Versión del SDK de .NET a instalar (ej. `10.0.x`)                    |
| `solution-path`   | sí        | `.sln`/`.slnx` a restaurar/compilar/empaquetar                       |
| `package-version` | sí        | Versión a estampar en cada paquete (ej. la de `semver-release.yml`)  |

## Consumir un feed NuGet privado

`dotnet-tests.yml`, `dotnet-publish-obfuscate.yml`, `nuget-vulnerability-scan.yml`
y `semver-release.yml` exportan siempre `NUGET_GITHUB_ACTOR`/`NUGET_GITHUB_TOKEN`
como variables de entorno (`NUGET_GITHUB_TOKEN` desde `secrets.NUGET_FEED_TOKEN`,
un secret que el repo consumidor debe definir — un PAT con scope
`read:packages`, ya que el `GITHUB_TOKEN` de un repo NO puede leer los
paquetes de otro). **No agregan ninguna fuente NuGet nueva** — eso ya lo
declara el propio `nuget.config` del repo consumidor (ver ejemplo abajo);
agregarla de nuevo desde acá chocaría con "the source specified has already
been added" apenas ese `nuget.config` ya la trajera consigo al hacer
checkout. Un repo sin ningún feed privado configurado simplemente no usa
esas dos variables — cero efecto.

Solo hace falta `secrets: inherit` en el caller (para que
`NUGET_FEED_TOKEN` llegue) y, en el propio repo, un `nuget.config` como:

```xml
<configuration>
  <packageSources>
    <add key="github" value="https://nuget.pkg.github.com/corebound-labs/index.json" />
  </packageSources>
  <packageSourceCredentials>
    <github>
      <add key="Username" value="%NUGET_GITHUB_ACTOR%" />
      <add key="ClearTextPassword" value="%NUGET_GITHUB_TOKEN%" />
    </github>
  </packageSourceCredentials>
</configuration>
```

```yaml
jobs:
  test:
    uses: corebound-labs/Commons-CI/.github/workflows/dotnet-tests.yml@master
    with:
      dotnet-version: '10.0.x'
      solution-path: 'EcoTrack.sln'
    secrets: inherit
```

Mismo `nuget.config` sirve para restaurar localmente (Visual Studio, `dotnet`
CLI) definiendo esas dos variables de entorno en la máquina (ej. `setx
NUGET_GITHUB_ACTOR "tu-usuario"` / `setx NUGET_GITHUB_TOKEN "ghp_..."`, y
reabrir la terminal/IDE para que las tome).

## Actions disponibles

### `.github/actions/obfuscate-js` — Minificar y ofuscar JS

Composite action (no workflow reutilizable: necesita operar sobre la carpeta
`publish` del propio job de deploy). Pasa cada `.js` por
[Terser](https://terser.org/) y luego por
[javascript-obfuscator](https://github.com/javascript-obfuscator/javascript-obfuscator), in situ.
No renombra globales, así que los handlers inline de las vistas siguen funcionando.

```yaml
- name: Ofuscar JS
  uses: corebound-labs/Commons-CI/.github/actions/obfuscate-js@master
  with:
    path: ${{ github.workspace }}/publish/wwwroot/js
```

| Input                | Requerido | Descripción                                             |
|----------------------|-----------|----------------------------------------------------------|
| `path`               | sí        | Carpeta con los `.js` a procesar                         |
| `exclude`            | no        | Patrones `find -name` a excluir, coma (def. `*.min.js`)  |
| `node-version`       | no        | Node.js (def. `22`)                                      |
| `terser-version`     | no        | terser (def. `5`)                                        |
| `obfuscator-version` | no        | javascript-obfuscator (def. `4`)                         |
| `allow-missing`      | no        | `true` no falla si `path` no existe, lo salta con aviso — para carpetas opcionales (def. `false`) |
| `allow-empty`        | no        | `true` no falla si `path` existe pero no tiene ningún `.js` (def. `false`) |

## Alcance

Este repo cubre reutilización de **pipelines** (YAML de GitHub Actions),
incluyendo empaquetar/publicar/consumir paquetes NuGet privados
(`dotnet-nuget-publish.yml`, `NUGET_GITHUB_ACTOR`/`NUGET_GITHUB_TOKEN`). El **código** C# compartido
en sí (`Commons.CrudOrm`, `Commons.Infisical`, etc.) vive en
[corebound-labs/Commons](https://github.com/corebound-labs/Commons), un repo
distinto — acá solo el pipeline que lo empaqueta/publica/permite consumir.
