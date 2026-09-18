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

---

# SECCIÓN 2 — Running Tests

## 2.1 Pruebas unitarias en Go

### Por qué importan en CI

Un pipeline de CI sin pruebas no verifica nada. El repo de Notely venía con **cero pruebas unitarias**, así que el primer paso fue escribirlas.

### El código bajo prueba: `internal/auth/auth.go`

```go
package auth

import (
	"errors"
	"net/http"
	"strings"
)

var ErrNoAuthHeaderIncluded = errors.New("no authorization header included")

// GetAPIKey -
func GetAPIKey(headers http.Header) (string, error) {
	authHeader := headers.Get("Authorization")
	if authHeader == "" {
		return "", ErrNoAuthHeaderIncluded
	}
	splitAuth := strings.Split(authHeader, " ")
	if len(splitAuth) < 2 || splitAuth[0] != "ApiKey" {
		return "", errors.New("malformed authorization header")
	}

	return splitAuth[1], nil
}
```

**Análisis:** recibe los headers HTTP, busca `Authorization`, espera el formato `ApiKey <valor>` y devuelve `<valor>`.

Tiene **tres caminos posibles** → mínimo tres casos de prueba:

| Camino | Entrada | Salida esperada |
|---|---|---|
| Éxito | `ApiKey mi-clave` | `"mi-clave"`, `nil` |
| Sin header | (vacío) | `""`, `ErrNoAuthHeaderIncluded` |
| Malformado | `Bearer mi-clave` | `""`, `errors.New("malformed authorization header")` |

### Reglas del lenguaje que condicionan el archivo de prueba

**1. El paquete lo define el directorio, no el archivo.**

Todos los archivos `.go` de un mismo directorio deben declarar el mismo `package X`.

```
internal/auth/                 ← este directorio = un paquete
├── auth.go                    → package auth
└── get_api_key_test.go        → package auth   (obligado)
```

Si no coinciden, el compilador falla:

```
found packages auth (auth.go) and split (get_api_key_test.go)
```

**2. El sufijo `_test.go` es aparte del `package`.**

Son dos cosas distintas que se confunden fácil:

| Elemento | Qué determina |
|---|---|
| `package auth` (dentro del archivo) | A qué paquete pertenece el archivo |
| `_test.go` (en el nombre del archivo) | Que es código de prueba → **se excluye del binario final** |

Equivale a tener `src/test/java` separado de `src/main/java` en Maven, pero resuelto por el nombre del archivo en lugar de por la carpeta.

**3. Única excepción — el paquete `_test`.**

Go permite un paquete extra por directorio: el que termina en `_test`. Es *black-box testing*, solo ve lo **exportado** (mayúscula inicial) y requiere import explícito:

```go
package auth_test

import (
	"testing"
	"github.com/bootdotdev/learn-cicd-starter/internal/auth"
)

func TestGetAPIKey(t *testing.T) {
	gotKey, gotErr := auth.GetAPIKey(...)   // requiere el prefijo
}
```

Para el curso conviene `package auth` a secas: menos ruido y permite probar funciones privadas.

### Equivalencias JUnit 5 ↔ Go

| Java / JUnit 5 | Go |
|---|---|
| `GetAPIKeyTest.java` | `get_api_key_test.go` — sufijo `_test.go` **obligatorio** |
| `@Test void shouldX()` | `func TestX(t *testing.T)` — prefijo `Test` **obligatorio** |
| `assertEquals(a, b)` | No existe: escribes un `if` y llamas `t.Errorf(...)` |
| `fail()` | `t.Fatalf(...)` |
| `@ParameterizedTest` / `@CsvSource` | *Table-driven tests*: `map` de casos + `for` |
| `mvn test` | `go test ./...` |
| `src/test/java` separado | Sufijo `_test.go` en el nombre |

**No hay librería de asserts en la stdlib.** Si nadie llama a `t.Errorf`, el test pasa.

### `t.Errorf` vs `t.Fatalf`

| | Comportamiento | Cuándo usarlo |
|---|---|---|
| `t.Errorf` | Registra el fallo y **continúa** | Comparaciones normales — muestra todos los problemas de una vez |
| `t.Fatalf` | Detiene el test **inmediatamente** | Cuando seguir causaría un *panic* (ej: desreferenciar un error `nil`) |

### Solución: `internal/auth/get_api_key_test.go`

```go
package auth

import (
	"errors"
	"net/http"
	"testing"
)

func TestGetAPIKey(t *testing.T) {
	tests := map[string]struct {
		headers http.Header
		wantKey string
		wantErr error
	}{
		"clave válida": {
			headers: http.Header{"Authorization": []string{"ApiKey mi-clave-secreta"}},
			wantKey: "mi-clave-secreta",
			wantErr: nil,
		},
		"sin header Authorization": {
			headers: http.Header{},
			wantKey: "",
			wantErr: ErrNoAuthHeaderIncluded,
		},
		"header malformado - prefijo incorrecto": {
			headers: http.Header{"Authorization": []string{"Bearer mi-clave-secreta"}},
			wantKey: "",
			wantErr: errors.New("malformed authorization header"),
		},
		"header malformado - sin valor": {
			headers: http.Header{"Authorization": []string{"ApiKey"}},
			wantKey: "",
			wantErr: errors.New("malformed authorization header"),
		},
	}

	for name, tc := range tests {
		t.Run(name, func(t *testing.T) {
			gotKey, gotErr := GetAPIKey(tc.headers)

			if gotKey != tc.wantKey {
				t.Errorf("clave: se obtuvo %q, se esperaba %q", gotKey, tc.wantKey)
			}

			if tc.wantErr == nil {
				if gotErr != nil {
					t.Errorf("error: se obtuvo %v, no se esperaba error", gotErr)
				}
				return
			}

			if gotErr == nil {
				t.Fatalf("error: no se obtuvo error, se esperaba %v", tc.wantErr)
			}

			if gotErr.Error() != tc.wantErr.Error() {
				t.Errorf("error: se obtuvo %q, se esperaba %q", gotErr, tc.wantErr)
			}
		})
	}
}
```

**Claves del código:**

- `struct{...}` anónimo dentro del `map` → la "fila" de la tabla de casos, equivalente a `@CsvSource`
- `http.Header` es un `map[string][]string`; por eso el valor va entre corchetes: `[]string{"ApiKey ..."}`
- `t.Run(name, func...)` crea un **subtest** con nombre propio: si falla, Go indica exactamente qué caso
- Los errores se comparan con `.Error()` (el texto) porque el error "malformed" se crea nuevo en cada llamada y no puede compararse por identidad con `==`

### Sobre `reflect.DeepEqual`

El [blog de Dave Cheney](https://dave.cheney.net/2019/05/07/prefer-table-driven-tests) que recomienda la lección usa `reflect.DeepEqual` porque compara un **slice** (`[]string`), y en Go los slices no se comparan con `==`.

Como `GetAPIKey` devuelve un `string`, basta con `!=` y no hace falta importar `reflect`.

⚠️ El código del blog **no funciona copiado literal**: declara `package split` y llama a una función `Split` que no existe en Notely. Del blog se toma el **patrón**, no el código.

### Verificación local

```bash
go test ./...                  # todos los tests
go test ./internal/auth -v     # detalle por subtest
go build ./...                 # compila (silencio = éxito)
go vet ./...                   # análisis estático
```

Salida esperada de `-v`:

```
=== RUN   TestGetAPIKey
=== RUN   TestGetAPIKey/clave_válida
=== RUN   TestGetAPIKey/sin_header_Authorization
=== RUN   TestGetAPIKey/header_malformado_-_prefijo_incorrecto
--- PASS: TestGetAPIKey (0.00s)
PASS
```

---

## 2.2 Tests on CI — ejecutar las pruebas en el pipeline

**Tarea:** eliminar el step `go version` del workflow y reemplazarlo por uno que ejecute las pruebas.

### Workflow actualizado: `.github/workflows/ci.yml`

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

      - name: Run unit tests
        run: go test ./...
```

El único cambio respecto a la Sección 1.4 es el último step: `run: go version` → `run: go test ./...`

### Por qué esto hace fallar el CI

Cada `run:` ejecuta un shell. Si el comando devuelve un **exit code distinto de 0**, el step falla y el job entero se marca en rojo.

`go test` devuelve `1` cuando algún test falla. Ahí está todo el mecanismo — no hay integración especial entre GitHub Actions y Go.

Esto conecta directamente con la convención de exit codes vista en la Sección 1.1.

### Validación del fallo (paso crítico de la lección)

La lección insiste en **romper el código a propósito** para confirmar que el CI realmente detecta fallos.

> *"Te sorprendería cuántas veces las empresas en las que he trabajado creían tener un CI que verificaba fallos, pero el código roto en realidad no hacía fallar el CI."*

Se modificó `internal/auth/auth.go` temporalmente:

```go
if len(splitAuth) < 2 || splitAuth[0] != "Bearer" {   // roto a propósito
```

Verificación del exit code — **esto es lo que lee GitHub Actions**:

```bash
go test ./... ; echo "exit code: $?"
```

```
--- FAIL: TestGetAPIKey/clave_válida (0.00s)
    get_api_key_test.go:39: clave: se obtuvo "", se esperaba "mi-clave-secreta"
FAIL
exit code: 1
```

Se hizo commit y push del código roto, se confirmó el ❌ en el PR, y luego se revirtió a `"ApiKey"` → ✅ verde.

### Diagrama del mecanismo completo

```
git push  →  evento pull_request  →  GitHub levanta un runner Ubuntu
                                      ├── actions/checkout  (clona el código)
                                      ├── actions/setup-go  (instala Go)
                                      └── go test ./...
                                             ├── exit 0 → ✅ step pasa → job verde
                                             └── exit 1 → ❌ step falla → job rojo
```

Un job se detiene en el **primer step que falle**; los siguientes no se ejecutan.

### Inspeccionar fallos desde la terminal

```bash
gh pr checks --watch        # seguir los checks en vivo
gh run list --limit 3
gh run view --log-failed    # logs de los steps que fallaron
```

---

## 2.3 Code Coverage

```
code_coverage = (lineas_cubiertas / lineas_totales) * 100
```

Si hay `1000` líneas de código y las pruebas cubren `500`, la cobertura es `50%`.

**Tarea:** agregar el flag `-cover` para imprimir la cobertura en los logs (sin hacer fallar el CI).

### El cambio

```yaml
      - name: Run unit tests
        run: go test -cover ./...
```

⚠️ **El orden importa.** En Go los flags van **entre el subcomando y los paquetes**. `go test ./... -cover` no funciona como se espera.

### Salida

```
ok      github.com/bootdotdev/learn-cicd-starter/internal/auth  0.003s  coverage: 100.0% of statements
?       github.com/bootdotdev/learn-cicd-starter               [no test files]
```

La línea con `?` significa que ese paquete **no tiene pruebas**. No cuenta como 0% — queda fuera del cálculo. Por eso aparece `100.0%`: es la cobertura del paquete `auth` solamente.

Para obtener el número global del proyecto:

```bash
go test -coverprofile=coverage.out ./...
go tool cover -func=coverage.out | tail -1
go tool cover -html=coverage.out          # reporte visual en el navegador
```

### Reportar vs. Exigir

| | Qué hace | Efecto |
|---|---|---|
| **Reportar** (`-cover`) | Imprime el % en los logs | Informativo. El CI pasa igual |
| **Exigir** (*quality gate*) | Compara el % contra un umbral | Si no llega, el build **falla** y el PR se bloquea |

Esta lección solo implementa el primero.

### Equivalencia con el stack de La Tinka (Java + SonarQube + JaCoCo)

```
mvn test  →  JaCoCo instrumenta y genera target/site/jacoco/jacoco.xml
                              ↓
          sonar-scanner lee ese XML
                              ↓
          SonarQube calcula % y lo compara con el Quality Gate
                              ↓
                  ✅ Passed  /  ❌ Failed
```

Equivalente local de `go test -cover`:

```bash
mvn clean verify
open target/site/jacoco/index.html
```

**Condiciones por defecto del *Sonar way*** — aplican sobre **código nuevo**, no sobre todo el proyecto (concepto *Clean as You Code*):

| Condición | Umbral |
|---|---|
| Cobertura en código nuevo | ≥ 80% |
| Líneas duplicadas en código nuevo | ≤ 3% |
| Bugs / vulnerabilidades nuevas | 0 |
| Security hotspots revisados | 100% |

**Cómo saber si el gate bloquea o solo reporta:**

1. Dashboard de SonarQube → *Project Settings → Quality Gate* (cuál está asignado) y *Quality Gates* global (sus condiciones)
2. En el pipeline, buscar el paso que espera el resultado. En Jenkins es:
   ```groovy
   waitForQualityGate abortPipeline: true
   ```
   Si ese paso **no está**, Sonar reporta pero no bloquea nada — el escenario más común en equipos que adoptaron Sonar sin cerrar el ciclo
3. `cat sonar-project.properties` → verificar que `sonar.coverage.jacoco.xmlReportPaths` apunte a un archivo que realmente se genera. Si no, Sonar reporta **0%** aunque existan pruebas

### Por qué la métrica es controversial

Es posible tener 100% de cobertura y aun así tener bugs, y 0% de cobertura con una app libre de errores. Las pruebas unitarias codifican el comportamiento esperado de unidades de código, pero no garantizan ausencia de bugs.

El autor del curso argumenta que no todas las funciones merecen la misma atención, y que **mockear sistemas externos (como bases de datos) en pruebas unitarias no es buena idea** — ese es mejor caso de uso para pruebas de integración.

Postura recomendada para un desarrollador que entra a un equipo nuevo: conocer la métrica, respetar el umbral de la organización, y plantear opiniones propias cuando ya se tenga confianza ganada.

---

## 2.4 README Badge

**Tarea:** agregar un badge dinámico al `README.md` que muestre el estado de las pruebas.

### Estructura de la URL

```
https://github.com/<OWNER>/<REPOSITORY>/actions/workflows/<WORKFLOW_FILE>/badge.svg
```

### Sintaxis de imagen en Markdown

```markdown
![texto alternativo](URL_DE_LA_IMAGEN)
```

### Línea agregada al inicio del README

```markdown
![Tests](https://github.com/cecibelauda/learn-cicd-starter/actions/workflows/ci.yml/badge.svg)
```

### Detalles importantes

| Aspecto | Detalle |
|---|---|
| Qué va en `<WORKFLOW_FILE>` | El **nombre del archivo** (`ci.yml`), no el `name:` interno |
| Qué texto muestra el badge | El `name:` interno del workflow → por eso dice `ci passing` |
| Es dinámico | GitHub regenera el SVG en cada request, consultando el último run |
| Qué rama consulta | La **rama por defecto** (`main`), salvo que se especifique otra |
| Cuándo aparece | Solo tras mergear a `main` — en una rama no se ve en la portada |

### Variantes útiles

```markdown
<!-- Badge de una rama específica -->
![Tests](https://github.com/cecibelauda/learn-cicd-starter/actions/workflows/ci.yml/badge.svg?branch=develop)

<!-- Badge clickeable → sintaxis [![alt](imagen)](destino) -->
[![Tests](https://github.com/cecibelauda/learn-cicd-starter/actions/workflows/ci.yml/badge.svg)](https://github.com/cecibelauda/learn-cicd-starter/actions/workflows/ci.yml)
```

### Si el badge sale gris con "no status"

Significa que el workflow nunca corrió sobre la rama por defecto. Con un `on:` que solo tiene `pull_request`, normalmente se resuelve tras el merge del PR.

Para forzar que también corra en `main`:

```yaml
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
```

**Resultado obtenido:** ✅ badge en verde mostrando `ci passing`.

---

## 2.5 Comandos nuevos de la Sección 2

### Go

```bash
go test ./...                    # todas las pruebas, recursivo
go test ./internal/auth -v       # detalle por subtest
go test -cover ./...             # con reporte de cobertura
go test ./... ; echo $?          # ver el exit code (0 = ok, 1 = falla)
go build ./...                   # compila (silencio = éxito)
go vet ./...                     # análisis estático
gofmt -w <archivo>               # formatea e indenta automáticamente

go test -coverprofile=coverage.out ./...
go tool cover -func=coverage.out | tail -1
go tool cover -html=coverage.out
```

`./...` significa: el directorio actual **y todos sus subdirectorios**.

### GitHub CLI

```bash
gh repo set-default cecibelauda/learn-cicd-starter   # obligatorio en forks
gh repo set-default --view

gh pr status
gh pr checks --watch
gh pr merge --merge
gh pr view --web

gh run list --limit 3
gh run view --log-failed
gh run watch
gh repo view --web
```

Alternativa sin configurar el default:

```bash
gh pr checks --repo cecibelauda/learn-cicd-starter
gh pr merge --repo cecibelauda/learn-cicd-starter --merge
```

### Ciclo de ramas por sección

```bash
git checkout main
git pull origin main             # ⚠️ el paso que más se olvida
git checkout -b <nombre-rama>
# ... trabajo ...
git add .
git commit -m "tipo: descripción"
git push origin <nombre-rama>
```

Limpieza tras el merge:

```bash
git branch -d <rama>                   # borra local
git push origin --delete <rama>        # borra remota
git fetch --prune                      # limpia referencias muertas
```

---

## 2.6 Notas y errores encontrados

| Situación | Causa | Solución |
|---|---|---|
| `zsh: command not found: code` | VS Code no instaló el comando en el PATH | En VS Code: `Cmd+Shift+P` → *Shell Command: Install 'code' command in PATH*. Alternativa: usar `nano` |
| `No default remote repository has been set` | `gh` detecta que el repo es un fork y no sabe si apuntar al fork o al upstream | `gh repo set-default cecibelauda/learn-cicd-starter` ⚠️ nunca `bootdotdev` |
| `found packages auth and split` | Se copió literal el código del blog de Dave Cheney | Usar `package auth` y llamar a `GetAPIKey`, no a `Split` |
| Código pegado en `nano` se escalona | Auto-indent del editor | Abrir con `nano -i`, o correr `gofmt -w <archivo>` después |
| `go test ./... -cover` no reporta cobertura | Los flags de Go van antes de los paquetes | `go test -cover ./...` |
| Badge en gris con "no status" | El workflow nunca corrió sobre `main` | Se resuelve tras el merge. Opcional: agregar `push: branches: [main]` |

### Reglas de trabajo consolidadas

1. **Una rama = una unidad de cambio = un PR.** Rama mergeada = ciclo cerrado, no reutilizar
2. **Siempre `git pull` en `main` antes de ramificar**, o habrá conflictos al mergear
3. **Validar el CI rompiéndolo a propósito** al menos una vez, para confirmar que sí detecta fallos
4. **Los flags de Go van antes de los paquetes:** `go test -cover ./...`
5. Tras pegar código Go en un editor de terminal, siempre `gofmt -w`

---

## 2.7 Flujo completo ejecutado en la Sección 2

```bash
# 1. Crear el archivo de pruebas
nano internal/auth/get_api_key_test.go     # contenido en la sección 2.1
gofmt -w internal/auth/get_api_key_test.go
go test ./...

# 2. Agregar las pruebas al workflow
nano .github/workflows/ci.yml              # go version → go test ./...

# 3. Romper el código a propósito y validar que el CI falla
nano internal/auth/auth.go                 # "ApiKey" → "Bearer"
go test ./... ; echo $?                    # debe imprimir 1
git add .
git commit -m "ci: run unit tests in CI (intentionally broken code)"
git push origin addtests
gh repo set-default cecibelauda/learn-cicd-starter
gh pr checks --watch                       # ❌ rojo

# 4. Arreglar el código
nano internal/auth/auth.go                 # "Bearer" → "ApiKey"
go test ./...
git add internal/auth/auth.go
git commit -m "fix: restore ApiKey prefix check"
git push origin addtests                   # re-dispara el workflow ✅

# 5. Agregar el flag de cobertura
nano .github/workflows/ci.yml              # go test -cover ./...
git add .github/workflows/ci.yml
git commit -m "ci: report test coverage"
git push origin addtests

# 6. Agregar el badge al README
nano README.md                             # línea del badge al inicio
git add README.md
git commit -m "docs: add tests status badge to README"
git push origin addtests

# 7. Mergear y sincronizar
gh pr checks --watch
gh pr merge --merge
git checkout main
git pull origin main
gh repo view --web                         # verificar badge "ci passing" ✅
```

---

## 2.8 Referencias oficiales de la Sección 2

**Go — testing**
- Paquete `testing`: https://pkg.go.dev/testing
- Tutorial oficial "Add a test": https://go.dev/doc/tutorial/add-a-test
- Flags de testing (incluye `-cover`): https://pkg.go.dev/cmd/go#hdr-Testing_flags
- Test packages: https://pkg.go.dev/cmd/go#hdr-Test_packages
- Package clause (spec): https://go.dev/ref/spec#Package_clause
- `reflect.DeepEqual`: https://pkg.go.dev/reflect#DeepEqual
- Dave Cheney — Prefer table driven tests: https://dave.cheney.net/2019/05/07/prefer-table-driven-tests

**GitHub Actions**
- `jobs.<job_id>.steps[*].run`: https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions#jobsjob_idstepsrun
- Adding a workflow status badge: https://docs.github.com/en/actions/monitoring-and-troubleshooting-workflows/monitoring-workflows/adding-a-workflow-status-badge

**GitHub CLI**
- `gh repo set-default`: https://cli.github.com/manual/gh_repo_set-default
- `gh pr merge`: https://cli.github.com/manual/gh_pr_merge

**Cobertura y calidad (contexto Java / La Tinka)**
- SonarQube — Quality Gates: https://docs.sonarsource.com/sonarqube-server/latest/instance-administration/analysis-functions/quality-gates/
- SonarQube — Test coverage: https://docs.sonarsource.com/sonarqube-server/latest/analyzing-source-code/test-coverage/overview/
- Boot.dev — Don't mock database connections: https://www.boot.dev/blog/backend/writing-good-unit-tests-dont-mock-database-connections/
- CircleCI — Unit vs integration testing: https://circleci.com/blog/unit-testing-vs-integration-testing/

**Markdown**
- Cheat sheet: https://www.markdownguide.org/cheat-sheet/
- Imágenes: https://www.markdownguide.org/basic-syntax/#images-1

---

# ESTADO DEL CURSO

| Sección | Estado |
|---|---|
| 1 — Fundamentos de CI/CD y GitHub Actions | ✅ Completada |
| 2 — Running Tests | ✅ Completada |
| 3 — Security | ⏳ Pendiente |
| 4 — Formatting / Linting | ⏳ Pendiente |
| 5 — Continuous Deployment | ⏳ Pendiente |

**Rama `addtests`:** mergeada a `main` ✅ — ciclo cerrado, crear rama nueva para la Sección 3.

**Próximo paso sugerido:**

```bash
git checkout main
git pull origin main
git checkout -b security
```
