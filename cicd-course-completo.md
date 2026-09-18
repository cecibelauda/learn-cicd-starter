# Curso: CI/CD with GitHub Actions — Boot.dev

**Estudiante:** Auda (Cecibel Espinoza)
**Repositorio:** https://github.com/cecibelauda/learn-cicd-starter
**Rama de trabajo:** `addtests`
**Directorio local:** `~/cicd_course/learn-cicd-starter`

---

# SECCIÓN 1 — Fundamentos de CI/CD y GitHub Actions

## 1.1 Conceptos base

### ¿Qué es CI/CD?

| Sigla | Significado | Qué hace |
|---|---|---|
| **CI** | Continuous Integration | Los devs suben cambios a un repo central y se disparan builds y pruebas automáticas |
| **CD** | Continuous Delivery / Deployment | Si CI pasa, el código se publica automáticamente |

**Tipos de pruebas que puede incluir el CI:**
- Pruebas unitarias
- Pruebas de integración
- Verificaciones de estilo (formatting)
- Linting
- Verificaciones de seguridad

Si alguna falla, el build se considera **"roto"** y se notifica al desarrollador.

> **Idea central:** CI se trata de automatizar la mayor parte posible del proceso de pruebas y revisión, para que el revisor humano no tenga que verificar formato ni correr tests localmente.

### Códigos de salida (exit codes)

Convención universal de las herramientas CLI:

| Exit code | Significado |
|---|---|
| `0` | Éxito — el step pasa |
| Cualquier otro | Falla — el step falla |

Ejemplo: `go test` sale con código `1` si un caso de prueba falla.

---

## 1.2 Flujo de trabajo con Git en equipo

### El problema del flujo lineal

```bash
# trabajando directo sobre "main"
git add .
git commit -m "mensaje"
git push origin main
```

Funciona para proyectos personales, pero en equipo genera problemas:
- Dos personas modifican la misma función y ambas pushean a `main`
- No hay punto de revisión de código antes del merge
- No se puede trabajar en varias funcionalidades en paralelo

### Ramas (branches)

Una **rama** es (básicamente) una copia de la base de código, de un tipo especial que hace simple fusionar cambios de una rama a otra.

**Convención en equipos:** `main` refleja el estado de producción → siempre debe estar estable y lista para desplegar.

Crear una rama nueva para:
- Agregar una funcionalidad
- Corregir un bug
- Refactorizar código

### Comandos de ramas

```bash
git branch                    # ver ramas; el asterisco marca la actual
git switch -c addtests        # crear rama nueva y cambiarse a ella
git switch addtests           # cambiarse a una rama existente
git push -u origin addtests   # subir la rama al remoto (solo existe local al crearla)
```

El flag `-u` establece el upstream: después basta con `git push`.

### Pull Requests

Un **PR** propone fusionar los cambios de una rama en otra. Permite revisión de código y ejecución de CI antes del merge.

**Crear un PR desde un fork — el error más común:**

GitHub pone por defecto el repositorio **original** como base repository. Hay que cambiarlo al fork propio.

URL directa para comparar dentro del propio fork:

```
https://github.com/cecibelauda/learn-cicd-starter/compare/main...addtests
```

Verificar que ambos lados digan `cecibelauda/learn-cicd-starter` (no `bootdotdev`).

Con GitHub CLI:

```bash
gh pr create --repo cecibelauda/learn-cicd-starter \
  --base main --head addtests \
  --title "Add tests" --body "Descripción"

gh pr list --repo cecibelauda/learn-cicd-starter   # verificar
gh pr close NUMERO --repo OWNER/REPO               # cerrar uno equivocado
```

---

## 1.3 Estructura de directorios `.github`

### Por qué el punto

El `.` inicial es una **convención de Unix**: marca archivos/directorios ocultos. No es de GitHub.

```bash
ls        # no muestra ocultos
ls -a     # muestra todo (-a = all)
```

Ejemplos que ya se usan a diario: `~/.zshrc`, `~/.gitconfig`, `~/.ssh/`, `.git/`

**En Finder (Mac):** `Cmd + Shift + .` alterna la vista de ocultos.

### El nombre `.github` sí es específico de GitHub

```
tu-repo/
├── .git/                     ← interno de Git (no se toca)
├── .github/                  ← convención de GitHub (sí se edita)
│   ├── workflows/
│   │   └── ci.yml            ← Actions busca AQUÍ y solo aquí
│   ├── PULL_REQUEST_TEMPLATE.md
│   ├── ISSUE_TEMPLATE/
│   ├── CODEOWNERS
│   └── dependabot.yml
├── .gitignore
├── README.md
└── main.go                   ← código
```

⚠️ Si el archivo está en `workflows/ci.yml` o en `.github/workflow/` (singular), **no se ejecuta nada**.

⚠️ Boot.dev valida la extensión `.yml` (no `.yaml`, aunque YAML acepte ambas).

### Regla general del punto

| Con punto | Sin punto |
|---|---|
| `.github/`, `.gitignore`, `.env`, `.dockerignore` | `README.md`, `LICENSE`, `Dockerfile`, `pom.xml` |
| Configuración de herramientas | Código, documentación, manifiestos |

Los directorios propios (`src/`, `internal/`, `docs/`) van **sin punto**.

---

## 1.4 Anatomía de un workflow de GitHub Actions

### Archivo completo de referencia

```yaml
name: ci

on:
  pull_request:
    branches: [main]

jobs:
  tests:
    name: Tests
    runs-on: ubuntu-latest

    steps:
      - name: Check out code
        uses: actions/checkout@v6

      - name: Set up Go
        uses: actions/setup-go@v6
        with:
          go-version: "1.27.1"

      - name: Echo Go version
        run: go version
```

### Jerarquía conceptual

```
Workflow  (el archivo ci.yml completo)
   │
   ├── se dispara por un EVENTO (on:)
   │
   └── Job(s)  (corren en un RUNNER = máquina virtual de GitHub)
          │
          └── Step(s)  (tareas individuales)
                 │
                 ├── uses: → ejecuta una ACTION reutilizable
                 └── run:  → ejecuta un comando de shell
```

### Desglose línea por línea

| Clave | Función |
|---|---|
| `name:` (nivel raíz) | Nombre legible del workflow |
| `on:` | Evento que dispara el workflow |
| `pull_request.branches: [main]` | Se dispara al abrir un PR hacia `main` |
| `jobs:` | Lista de jobs que componen el workflow |
| `runs-on:` | Tipo de runner (VM). `ubuntu-latest` = última versión de Ubuntu |
| `steps:` | Lista de tareas del job |
| `uses:` | Action reutilizable a ejecutar |
| `with:` | Inputs (parámetros) de la action |
| `run:` | Comando de línea de comandos arbitrario en el runner |

### Conceptos clave

- **Workflow:** se dispara cuando ocurre un evento en el repositorio.
- **Job:** conjunto de steps que corren en el mismo runner. Se usan varios jobs para correr pruebas en paralelo o sobre múltiples sistemas operativos.
- **Runner:** máquina virtual en los servidores de GitHub que ejecuta el job.
- **Step:** tarea única — un comando, un script o una action.
- **Action:** aplicación personalizada reutilizable que reduce la complejidad de crear workflows.

### Actions usadas

| Action | Para qué |
|---|---|
| `actions/checkout@v6` | Clona el repo dentro del runner. **Casi siempre necesaria** |
| `actions/setup-go@v6` | Configura el entorno de Go |

### Comportamiento del re-disparo

Un workflow que se dispara con `pull_request` **se vuelve a ejecutar automáticamente** cuando se actualiza la rama a fusionar. No hace falta cerrar y reabrir el PR.

---

## 1.5 Comandos de terminal aprendidos

### Crear directorios y archivos

```bash
mkdir docs                 # un directorio
mkdir docs scripts         # varios
mkdir -p .github/workflows # anidados, crea los padres faltantes

touch notas.md             # archivo vacío
```

### Ver vs. editar archivos

`cat` solo **muestra**, no edita.

```bash
cat README.md              # mostrar contenido
nano README.md             # editar (Ctrl+O guardar, Ctrl+X salir)
vim README.md              # editar (:wq guardar y salir)
code README.md             # abrir en VS Code
```

### Redirecciones

```bash
echo "texto" >> README.md  # AGREGA al final
echo "texto" >  README.md  # SOBRESCRIBE todo (¡cuidado!)
```

Heredoc para escribir varias líneas:

```bash
cat > .github/workflows/ci.yml << 'EOF'
name: ci
...
EOF
```

Las comillas simples en `'EOF'` evitan que el shell interprete `$` o backticks.

### Verificación

```bash
tail -5 README.md          # últimas 5 líneas
ls -la                     # listado detallado con ocultos
git diff                   # qué cambió desde el último commit
git status                 # estado del working tree
```

En `ls -la`: los directorios empiezan con `d`, los archivos con `-`.

```
drwxr-xr-x  2 auda  staff   64 Sep 15 10:22 docs
-rw-r--r--  1 auda  staff    0 Sep 15 10:22 notas.md
```

### Verificar workflows con GitHub CLI

```bash
gh run list --branch addtests --limit 5   # últimas ejecuciones
gh run view --log-failed                  # logs de los steps fallidos
```

---

## 1.6 Notas y errores encontrados

| Situación | Causa | Solución |
|---|---|---|
| PR abierto contra `bootdotdev` en vez del fork | GitHub pone el repo original como base por defecto | Cerrar el PR y usar la URL `compare` del propio fork |
| `cat` no permite editar | `cat` es solo de lectura | Usar `nano`, `vim`, `code` o redirección `>>` |
| Git no versiona directorios vacíos | Comportamiento de diseño de Git | Agregar un archivo dentro, o `touch docs/.gitkeep` |
| Workflow no aparece en el PR | Ruta incorrecta o PR contra el repo equivocado | Verificar `.github/workflows/` exacto |
| Error "workflow file issue" | Indentación YAML con tabs | YAML solo acepta **espacios**, nunca tabs |
| `setup-go` no encuentra la versión | Versión no disponible en el runner | Alinear con `go.mod` o usar `go-version-file: go.mod` |

---

## 1.7 Flujo completo ejecutado en la Sección 1

```bash
# 1. Fork del repo en GitHub (botón Fork)

# 2. Clonar el fork
git clone https://github.com/cecibelauda/learn-cicd-starter.git
cd learn-cicd-starter

# 3. Crear rama de trabajo
git branch
git switch -c addtests
git push -u origin addtests

# 4. Editar README y commitear
echo "Cecibel's version of Boot.dev's Notely app." >> README.md
git add README.md
git commit -m "update README"
git push origin addtests

# 5. Abrir PR (sin fusionar) — base: main, compare: addtests, ambos en el fork

# 6. Crear el workflow de CI
mkdir -p .github/workflows
nano .github/workflows/ci.yml      # contenido en la sección 1.4

# 7. Commit y push → el CI se dispara en el PR
git add .github/workflows/ci.yml
git commit -m "add ci workflow"
git push origin addtests

# 8. Verificar
gh run list --branch addtests --limit 3
```

---

## 1.8 Referencias oficiales

**GitHub Actions**
- Documentación principal: https://docs.github.com/en/actions
- Entender GitHub Actions: https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions
- Sintaxis de workflows: https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions
- Eventos que disparan workflows: https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows
- Exit codes en actions: https://docs.github.com/en/actions/creating-actions/setting-exit-codes-for-actions

**Actions usadas**
- `actions/checkout`: https://github.com/actions/checkout
- `actions/setup-go`: https://github.com/actions/setup-go

**Git / GitHub**
- Fork a repo: https://docs.github.com/en/get-started/quickstart/fork-a-repo
- Sobre las ramas: https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/about-branches
- PR desde un fork: https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/creating-a-pull-request-from-a-fork
- Archivos de comunidad en `.github`: https://docs.github.com/en/communities/setting-up-your-project-for-healthy-contributions/creating-a-default-community-health-file

**Otros**
- YAML: https://en.wikipedia.org/wiki/YAML
- `nano`: https://www.nano-editor.org/dist/latest/nano.html
- Git FAQ (directorios vacíos): https://git-scm.com/docs/gitfaq#empty-directories
