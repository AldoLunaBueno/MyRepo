# GitHub CLI y GitHub Actions

## Autenticación

```bash
echo "${{ secrets.GH_TOKEN }}" | gh auth login --with-token
```

Aquí, el token es enviado por ``echo`` a través de una tubería (|), lo que significa que el token se pasa como entrada estándar (stdin) al comando, como si el usuario lo escribiera después de ejecutar el comando.

```bash
gh pr view [PR_NUMBER] --json closingIssuesReferences
```

```json
{
  "closingIssuesReferences": [
    {
      "id": "I_kwDOOqr6t8621N9V",
      "number": 2,
      "repository": {
        "id": "R_kgDOOqr6tw",
        "name": "MyRepo",
        "owner": {
          "id": "U_kgDOBtYHfA",
          "login": "AldoLunaBueno"
        }
      },
      "url": "https://github.com/AldoLunaBueno/MyRepo/issues/2"
    }
  ]
}
```

![alt](images/2025-05-16-13-38-03.png)

Para esta parte se estuvo haciendo pruebas en [un repositorio](https://github.com/AldoLunaBueno/MyRepo) y [un proyecto](https://github.com/users/AldoLunaBueno/projects/5/views/1) aparte, los cuales van a ser referenciados en todo el ejercicio.

Se codificó [un flujo de trabajo](.github/workflows/move-to-in-progress.yml) para el caso de mover una historia de usuario a la columna "In Progress" cuando se asocia una pull request.

La acción se desencadena cuando una PR es abierta o reabierta para su revisión:

```yml
on:
	pull_request:
        types: [opened, reopened]
```

La herramienta clave que se usó en todos los pasos de ejecución fue GitHub CLI. Esta herramienta viene integrada en el runner de GitHub Actions. 

Para usarla en el flujo es necesario autenticarse con ``gh auth login``. Una opción es usar un token personal de acceso con permisos (scopes) de repositorio, proyecto, admin y workflow. Este token se guarda como un secreto de repositorio (GH_TOKEN) en GitHub Secrets y se accede a este desde el entorno del runner con ``secrets.GH_TOKEN``. La autenticación queda así en el paso de autenticación:

```bash
echo "${{ secrets.GH_TOKEN }}" | gh auth login --with-token
```

> El problema de usar un token de acceso personal es que no hay trazabilidad de sus premisos. Una mejor opción para este ejercicio habría sido generar [un token automático](https://docs.github.com/es/actions/writing-workflows/workflow-syntax-for-github-actions#defining-access-for-the-github_token-scopes) generado desde adentro del runner de GitHub Actions. Su referencia habría sido GITHUB_TOKEN.

Pasando al paso principal, para mover una historia a la columna "In Progress" usamos este comando de GitHub CLI:

```bash
gh project item-edit --project-id "$PROJECT_ID" --id "$ITEM_ID" \
    --field-id "$FIELD_ID" --single-select-option-id "$OPTION_ID"
```

Pero este comando requiere que obtengamos previamente otros valores:

- ID del item que representa la issue que guarda la historia de usuario
- ID del proyecto
- ID del campo Status que guarda todos los estados de la tabla Kanban
- ID de la opción de selección única a la que se va a mover el item, es decir, columna "In Progress"

Y cada una de estas ID se obtiene por medio de consultas que también se hacen usando comandos de GitHub CLI usando [la sintaxis jq](https://jqlang.org/manual/). 

Por ejemplo, para obtener todos los proyectos se aplica fácilmente el comando ``gh project list`` y se obtiene una respuesta en forma de tabla:

```txt
NUMBER  TITLE                          STATE  ID                  
5       Automated Project Test         open   PVT_kwHOBtYHfM4A5HUu
3       @AldoLunaBueno's Kanban Board  open   PVT_kwHOBtYHfM4A351k
```

Si sabemos que el proyecto al cual apuntamos es el número 5, podemos copiar manualmente la ID correspondiente 'PVT_kwHOBtYHfM4A5HUu'. Pero ¿cómo se hace esto la herramienta GitHub CLI? Primero la salida debe estar en formato json, para lo cual usamos la flag ``--format``:

```bash
gh project list --format json
```

Salida:

```json
{
  "projects": [
    {
      "closed": false,
      "fields": {
        "totalCount": 10
      },
      "id": "PVT_kwHOBtYHfM4A5HUu",
      "items": {
        "totalCount": 3
      },
      "number": 5,
      "owner": {
        "login": "AldoLunaBueno",
        "type": "User"
      },
      "public": true,
      "readme": "",
      "shortDescription": "",
      "title": "Automated Project Test",
      "url": "https://github.com/users/AldoLunaBueno/projects/5"
    },
	...
```

Y esta salida en formato json se filtra con una consulta jq incorporada en GitHub CLI con la flag --jq o solo -q:

```bash
PROJECT_ID=$(gh project list --format json \
    -q ".projects[]|select(.number == 5).id")
```

Lo que hace este filtro es seleccionar de entre todos los proyectos solo el que tenga una clave "number" con el valor 5, que es el número del proyecto al que apuntamos, y de toda la información del proyecto solo se toma su ID.

Algunos valores se guardaron en la parte `env` del paso principal en claves (en mayúsculas):

```yml
env:
	PR_NUMBER: ${{ github.event.pull_request.number }}
	PROJECT_NUMBER: 5 # project number asociated with repo
	ORG: "@me" # owner of the personal authentication token (for project, not PR)
	STATUS_NAME: "In Progress"
```

Se accede a estas claves lo largo de la ejecución del paso principal usando el símbolo de dolar ($) como cualquier variable de Bash.
