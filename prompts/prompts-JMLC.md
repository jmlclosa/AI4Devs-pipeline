# Prompts usando Jetbrains AI Assistant en IntelliJ

## Prompt para pedir una acción con los pasos checkout + test + build

```
Actúa como un devops experto en Github actions.
Necesito implementar una Github action en el fichero #file:ci.yml  que se ejecute cuando se haga un push en una rama con un Pull Request abierto

Los pasos que se deben seguir son:
1. Checkout
3. Ejecutar tests del módulo backend
4. Generar una build del módulo backend

Puedes utilizar el fichero #file:README.md  para saber como se compila y lanza el proyecto.

Sigue buenas prácticas en todo momento de seguridad, legibilidad, etc.
```

### Resultado

```yaml
name: CI Pipeline

on:
  push:
    branches:
      - '**'

jobs:
  build-and-test:
    if: github.event.pull_request != null
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout repository
      uses: actions/checkout@v2

    - name: Set up Node.js
      uses: actions/setup-node@v2
      with:
        node-version: '18'

    - name: Install dependencies (backend)
      working-directory: ./backend
      run: npm install

    - name: Run tests (backend)
      working-directory: ./backend
      run: npm test
    
    - name: Build backend
      working-directory: ./backend
      run: npm run build

  publish-artifacts:
    if: github.event.pull_request != null
    needs: build-and-test
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout repository
      uses: actions/checkout@v2

    - name: Upload Backend Build Artifacts
      uses: actions/upload-artifact@v3
      with:
        name: backend-build
        path: ./backend/build
```

Falla al ejecutarse en Github porque la acción tiene versiones deprecadas desde 2023.

## Prompt para corregir error

```
Al ejecutarse la acción en Github se produce un error porque se ha deprecado `actions/upload-artifact@v2`.
Utiliza el siguiente enlace para revisarlo:
https://github.blog/changelog/2024-02-13-deprecation-notice-v1-and-v2-of-the-artifact-actions/ 
```

> Gracias por la observación. Según el enlace proporcionado, las versiones v1 y v2 de las acciones de artefactos se han deprecado. Vamos a actualizar la acción actions/upload-artifact a la versión v3

### Resultado



Sigue sin ser la última versión de la acción y salta un warning en Github.

Además no parece encontrar los artefactos generados:

> publish-artifacts
> No files were found with the provided path: ./backend/build. No artifacts will be uploaded.


## Probamos con Claude 3.5 Sonnet desde su Chat (sin contexto del proyecto)

```
Actúa como un devops experto en Github actions.
Necesito implementar una Github action en el fichero #file:ci.yml  que se ejecute cuando se haga un push en una rama con un Pull Request abierto
Los pasos que se deben seguir son:
1. Checkout
3. Ejecutar tests del módulo backend
4. Generar una build del módulo backend
Puedes utilizar el fichero #file:README.md  para saber como se compila y lanza el proyecto.
Sigue buenas prácticas en todo momento de seguridad, legibilidad, etc.
```

### Resultado

```yaml
name: CI Pipeline

on:
  push:
    branches:
      # Se ejecuta en pushes a ramas que tienen PR abiertos
      - '**'
    paths:
      # Solo se ejecuta si hay cambios en el backend
      - 'backend/**'
      - '.github/workflows/**'

# Establecer permisos mínimos necesarios siguiendo el principio de menor privilegio
permissions:
  contents: read
  checks: write

jobs:
  backend-ci:
    name: Backend CI
    runs-on: ubuntu-latest
    
    # Prevenir ejecuciones concurrentes en el mismo PR/branch
    concurrency:
      group: ${{ github.workflow }}-${{ github.ref }}
      cancel-in-progress: true

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          # Fetch solo el commit necesario para optimizar
          fetch-depth: 1
          # Verificar la firma del commit por seguridad
          persist-credentials: false

      # Cache de dependencias para optimizar builds
      - name: Cache dependencies
        uses: actions/cache@v3
        with:
          path: |
            ~/.npm
            ~/.cache
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
          
      - name: Install dependencies
        working-directory: ./backend
        run: |
          npm ci
        env:
          # Reduce el output de npm para logs más limpios
          npm_config_loglevel: error

      - name: Run tests
        working-directory: ./backend
        run: npm test
        env:
          # Variables de entorno para tests si son necesarias
          NODE_ENV: test

      - name: Build backend
        working-directory: ./backend
        run: npm run build
        env:
          # Variables de entorno para build si son necesarias
          NODE_ENV: production

      # Guardar los artifacts de build para posible uso posterior
      - name: Upload build artifacts
        uses: actions/upload-artifact@v3
        with:
          name: backend-build
          path: backend/dist/
          retention-days: 7
```

Pipeline mucho más completa con: límite de paths, permisos, gestión de concurrencia de jobs y caché de npm; además de usar versiones más recientes de las actions.

## Pruebo con ChatGPT o1-mini (gratuito)

### Resultado
```yaml
name: CI for Backend Module

on:
  push:
    branches-ignore:
      - main
  pull_request:
    branches:
      - '*'

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      # Step 1: Checkout the code
      - name: Checkout code
        uses: actions/checkout@v2
      
      # Step 2: Set up Node.js
      - name: Set up Node.js
        uses: actions/setup-node@v2
        with:
          node-version: '16'  # Usar la versión recomendada en el proyecto

      # Step 3: Install dependencies
      - name: Install dependencies
        working-directory: ./backend
        run: npm install

      # Step 4: Run tests
      - name: Run tests
        working-directory: ./backend
        run: npm run test

      # Step 5: Build the project
      - name: Build the project
        working-directory: ./backend
        run: npm run build

```

Más sencillo que Claude, similar al de Jetbrains AI.

## Prueba con Cursor con Claude 3.5

### Resultado
```yaml
name: Pull Request CI

on:
  push:
    branches:
      - '**'
    paths:
      - 'backend/**'
  pull_request:
    paths:
      - 'backend/**'

jobs:
  backend-ci:
    name: Backend CI
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: ./backend

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
          cache-dependency-path: './backend/package-lock.json'

      - name: Install dependencies
        run: npm ci
        
      - name: Run tests
        run: npm test
        env:
          NODE_ENV: test
          
      - name: Build backend
        run: npm run build
```

# Iteración 2 sobre el pipeline ofrecido por Claude 3.5 chat

## Prompt para consultar donde ver el artefacto

```
Dónde debería ver el artefacto publicado?
```

### Respuesta
```
En GitHub Actions, los artefactos publicados se pueden encontrar en varios lugares:

En la interfaz web de GitHub:

Ve a tu repositorio
Haz clic en la pestaña "Actions"
Selecciona la ejecución específica del workflow
Desplázate hacia abajo hasta la sección "Artifacts"
Verás "backend-build" listado ahí, que puedes descargar como un archivo ZIP


Vía API: También puedes acceder programáticamente usando la GitHub REST API.

...
```

## Prompt para saber más sobre la caché del pipeline

```
Puedes explicar por qué en el paso "Cache dependencies" son distintas la key y restore-keys?
```

```
Y qué tiene que ver el paso "Cache dependencies" con el paso "Setup Node.js" que contiene un atributo "cache: 'npm'"?
```

### Respuesta

¡Excelente pregunta! En realidad, estamos duplicando la funcionalidad de caché, lo cual no es una buena práctica. 

El atributo `cache: 'npm'` en `actions/setup-node` ya implementa automáticamente el cacheo de dependencias npm de una manera optimizada. Veamos la diferencia:

1. **actions/setup-node con cache: 'npm'**:
```yaml
- name: Setup Node.js
  uses: actions/setup-node@v4
  with:
    node-version: '18'
    cache: 'npm'  # Maneja automáticamente el cacheo de node_modules
```
- Está específicamente diseñado para npm
- Maneja automáticamente el hash del package-lock.json
- Se integra mejor con npm ci
- Es más simple y menos propenso a errores

2. **actions/cache manual** (el que deberíamos eliminar):
```yaml
- name: Cache dependencies
  uses: actions/cache@v3
  with:
    path: |
      ~/.npm
      ~/.cache
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-
```
- Es más genérico
- Requiere configuración manual
- Podría interferir con el cacheo automático

Vamos a corregir el workflow eliminando la duplicación:



```yaml
name: CI Pipeline

on:
  push:
    branches:
      - '**'
    paths:
      - 'backend/**'
      - '.github/workflows/**'

permissions:
  contents: read
  checks: write

jobs:
  backend-ci:
    name: Backend CI
    runs-on: ubuntu-latest
    
    concurrency:
      group: ${{ github.workflow }}-${{ github.ref }}
      cancel-in-progress: true

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 1
          persist-credentials: false

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
          cache-dependency-path: './backend/package-lock.json'  # Especificamos la ruta al package-lock.json
          
      - name: Install dependencies
        working-directory: ./backend
        run: |
          npm ci
        env:
          npm_config_loglevel: error

      - name: Run tests
        working-directory: ./backend
        run: npm test
        env:
          NODE_ENV: test

      - name: Build backend
        working-directory: ./backend
        run: npm run build
        env:
          NODE_ENV: production

      - name: Upload build artifacts
        uses: actions/upload-artifact@v3
        with:
          name: backend-build
          path: backend/dist/
          retention-days: 7

      - name: Display artifact information
        run: |
          echo "✅ Build artifact has been published!"
          echo "📁 Artifact name: backend-build"
          echo "📍 Location: GitHub Actions > This workflow run > Artifacts section"
          echo "⏳ Retention period: 7 days"
          echo "🔍 You can find this artifact at:"
          echo "   ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"

```

Cambios realizados:
1. Eliminé el paso redundante de `Cache dependencies`
2. Añadí `cache-dependency-path` en setup-node para especificar la ubicación del package-lock.json
3. Mantuve el resto del workflow igual

Beneficios de este cambio:
- Código más limpio y mantenible
- Menos posibilidad de conflictos de caché
- Mejor integración con npm
- Mejor rendimiento al evitar operaciones duplicadas

¿Te gustaría que profundicemos en cómo funciona internamente el cacheo de setup-node o que ajustemos alguna otra configuración?


# 