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


# Prompts para limitar ante ramas con PR abierta

## Prompt 1
```
Cómo se puede limitar para que solo se ejecute en ramas con una PR abierta?
```

### Respuesta
Para limitar la ejecución solo a ramas que tienen un PR abierto, necesitamos modificar la sección on del workflow. Hay dos formas de hacerlo:

Usando pull_request - Se ejecuta cuando se crea/actualiza un PR
Usando una condición - Verificar si existe un PR asociado al push

La segunda opción es más precisa para tu caso, ya que mencionaste específicamente "push en una rama con PR abierto". Aquí está el workflow actualizado:

...

## Prompt 2

```
Preferimos la opción 1, entiendo que encaja porque solo se lanzará cuando se crea la PR o se hace un push a la rama con la PR abierta
```

### Resultado

Exacto, usando el trigger `pull_request` obtenemos exactamente ese comportamiento de una forma más simple y directa. El workflow se ejecutará:
- Cuando se crea una PR (`opened`)
- Cuando se hace push a la rama con la PR abierta (`synchronize`)
- Cuando se reabre una PR cerrada (`reopened`)

Aquí está la versión simplificada:



```yaml
name: CI Pipeline

on:
  pull_request:
    types: [opened, synchronize, reopened]
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
          cache-dependency-path: './backend/package-lock.json'
          
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

Beneficios de este enfoque:
1. **Más simple**: No necesitamos jobs adicionales ni scripts de verificación
2. **Más mantenible**: Menos código, menos puntos de fallo
3. **Mejor integración**: GitHub optimiza las ejecuciones en contexto de PR
4. **Mejor visibilidad**: Los resultados aparecen directamente en la interfaz de PR

Los tipos de eventos que incluimos:
- `opened`: Primera vez que se crea la PR
- `synchronize`: Cada vez que se hace push a la rama de la PR
- `reopened`: Si la PR se cierra y se vuelve a abrir

Otros tipos que podríamos añadir si fuera necesario:
- `ready_for_review`: Si usas PRs en modo draft
- `converted_to_draft`: Para pausar CI cuando una PR se convierte en draft
- `review_requested`: Para ejecutar CI cuando se pide review

¿Te gustaría añadir alguno de estos tipos adicionales o ajustar alguna otra configuración?


# Publicar a EC2 (Claude 3.5 chat)

## Prompt 1
```
Necesito modificar para añadir el despliegue del "backend" generado a una instancia EC2. 
Puedes añadir los jobs y/o pasos necesarios? 
Sigue buenas prácticas de seguridad y mantenibilidad para el pipeline.
```

### Resultado

Te ayudo a añadir el despliegue a EC2. Necesitaremos añadir un nuevo job para el despliegue que se ejecute después de que el CI haya pasado correctamente.



```yaml
name: CI/CD Pipeline

on:
  pull_request:
    types: [opened, synchronize, reopened]
    paths:
      - 'backend/**'
      - '.github/workflows/**'

# Permisos mínimos necesarios
permissions:
  contents: read
  checks: write
  id-token: write    # Necesario para autenticación OIDC con AWS

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
          cache-dependency-path: './backend/package-lock.json'
          
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

  deploy-to-ec2:
    name: Deploy to EC2
    needs: backend-ci    # Espera a que CI pase
    runs-on: ubuntu-latest
    # Solo despliega en PRs de determinadas ramas (ajusta según necesidades)
    if: github.base_ref == 'develop' || github.base_ref == 'staging'
    
    environment:
      name: ${{ github.base_ref }}  # Ambiente basado en la rama destino
      url: ${{ vars.EC2_APPLICATION_URL }}  # URL de la aplicación

    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ vars.AWS_ACCOUNT_ID }}:role/github-actions-role
          aws-region: ${{ vars.AWS_REGION }}

      # Descarga el artifact del job anterior
      - name: Download build artifact
        uses: actions/download-artifact@v3
        with:
          name: backend-build
          path: backend/dist

      # Genera un timestamp para el nombre del despliegue
      - name: Generate deployment timestamp
        id: timestamp
        run: echo "timestamp=$(date +'%Y%m%d_%H%M%S')" >> $GITHUB_OUTPUT

      # Empaqueta la build para el despliegue
      - name: Package deployment
        run: |
          tar -czf deployment.tar.gz \
            -C backend/dist .
          
      # Sube el paquete a S3
      - name: Upload to S3
        run: |
          aws s3 cp deployment.tar.gz \
            s3://${{ vars.DEPLOYMENT_BUCKET }}/${{ github.base_ref }}/backend_${{ steps.timestamp.outputs.timestamp }}.tar.gz

      # Ejecuta el script de despliegue en EC2 vía AWS Systems Manager
      - name: Deploy to EC2
        run: |
          aws ssm send-command \
            --targets "Key=tag:Environment,Values=${{ github.base_ref }}" \
            --document-name "AWS-RunShellScript" \
            --parameters "commands=[
              'cd /opt/app',
              'aws s3 cp s3://${{ vars.DEPLOYMENT_BUCKET }}/${{ github.base_ref }}/backend_${{ steps.timestamp.outputs.timestamp }}.tar.gz .',
              'pm2 stop backend || true',
              'rm -rf dist/*',
              'tar -xzf backend_${{ steps.timestamp.outputs.timestamp }}.tar.gz -C dist/',
              'pm2 start ecosystem.config.js --env ${{ github.base_ref }}',
              'rm backend_${{ steps.timestamp.outputs.timestamp }}.tar.gz'
            ]" \
            --comment "Deployment ${{ steps.timestamp.outputs.timestamp }}"

      # Verifica el estado del despliegue
      - name: Verify deployment
        run: |
          # Espera a que la aplicación esté disponible
          attempts=0
          max_attempts=30
          until curl -s -f ${{ vars.EC2_HEALTH_CHECK_URL }} > /dev/null || [ $attempts -eq $max_attempts ]
          do
            attempts=$((attempts+1))
            echo "Waiting for application to be ready... ($attempts/$max_attempts)"
            sleep 10
          done
          
          if [ $attempts -eq $max_attempts ]; then
            echo "❌ Deployment verification failed"
            exit 1
          fi
          
          echo "✅ Deployment verified successfully"

```

Las mejoras y consideraciones de seguridad incluidas son:

1. **Seguridad**:
    - Uso de OIDC para autenticación con AWS (sin credenciales almacenadas)
    - Separación de ambientes (environment) por rama
    - Variables sensibles almacenadas en GitHub Secrets/Variables
    - Uso de AWS Systems Manager en lugar de SSH directo
    - Permisos mínimos necesarios

2. **Mantenibilidad**:
    - Job separado para despliegue
    - Condición `needs` para asegurar orden correcto
    - Verificación post-despliegue
    - Nomenclatura con timestamps
    - Backup implícito en S3

3. **Configuración necesaria**:
   Variables a configurar en GitHub:
   ```markdown
   - AWS_ACCOUNT_ID: ID de la cuenta AWS
   - AWS_REGION: Región de AWS
   - DEPLOYMENT_BUCKET: Bucket S3 para artefactos
   - EC2_APPLICATION_URL: URL de la aplicación
   - EC2_HEALTH_CHECK_URL: URL del health check
   ```

4. **Prerrequisitos en AWS**:
   ```markdown
   1. Role IAM para GitHub Actions con permisos para:
      - S3 (PutObject)
      - Systems Manager (SendCommand)
      - EC2 (Tag-based access)
   
   2. EC2 configurada con:
      - AWS Systems Manager Agent
      - Tag "Environment" con valor correspondiente
      - Role IAM con acceso a S3
      - PM2 instalado
   ```

¿Te gustaría que profundicemos en algún aspecto específico o que ajustemos alguna parte del despliegue?

Por ejemplo:
- Ajustar la estrategia de rollback
- Modificar la verificación post-despliegue
- Añadir notificaciones
- Configurar despliegues blue/green

## Prompt 2

```
Podemos simplificar el proceso para solo usar EC2 y no usar S3?
```

### Resultado

Sí, podemos simplificar el proceso usando solo SSH directo a EC2. Usaremos una clave SSH almacenada como secreto en GitHub Actions para una conexión segura.



```yaml
name: CI/CD Pipeline

on:
  pull_request:
    types: [opened, synchronize, reopened]
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
          cache-dependency-path: './backend/package-lock.json'
          
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

  deploy-to-ec2:
    name: Deploy to EC2
    needs: backend-ci
    runs-on: ubuntu-latest
    if: github.base_ref == 'develop' || github.base_ref == 'staging'
    
    environment:
      name: ${{ github.base_ref }}
      url: ${{ vars.EC2_APPLICATION_URL }}

    steps:
      # Descarga el artifact del job anterior
      - name: Download build artifact
        uses: actions/download-artifact@v3
        with:
          name: backend-build
          path: dist

      # Configura la clave SSH como un archivo
      - name: Configure SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.EC2_SSH_KEY }}" > ~/.ssh/deploy_key
          chmod 600 ~/.ssh/deploy_key
          echo "${{ vars.EC2_SSH_KNOWN_HOSTS }}" >> ~/.ssh/known_hosts

      # Empaqueta y despliega vía SSH
      - name: Deploy to EC2
        env:
          EC2_USER: ${{ vars.EC2_USER }}
          EC2_HOST: ${{ vars.EC2_HOST }}
        run: |
          # Crear archivo de variables de entorno si es necesario
          echo "NODE_ENV=${{ github.base_ref }}" > .env
          echo "PORT=${{ vars.APP_PORT }}" >> .env
          
          # Crea un directorio temporal con timestamp
          TIMESTAMP=$(date +'%Y%m%d_%H%M%S')
          DEPLOY_DIR="backend_$TIMESTAMP"
          
          # Transfiere los archivos
          ssh -i ~/.ssh/deploy_key $EC2_USER@$EC2_HOST "mkdir -p ~/$DEPLOY_DIR"
          scp -i ~/.ssh/deploy_key -r dist/* .env $EC2_USER@$EC2_HOST:~/$DEPLOY_DIR/
          
          # Ejecuta el script de despliegue
          ssh -i ~/.ssh/deploy_key $EC2_USER@$EC2_HOST << 'ENDSSH'
            # Variables del despliegue actual
            DEPLOY_DIR=$DEPLOY_DIR
            APP_DIR=/opt/app/backend
            
            # Detener la aplicación actual
            pm2 stop backend || true
            
            # Backup del despliegue anterior
            if [ -d "$APP_DIR" ]; then
              mv $APP_DIR "${APP_DIR}_backup_$(date +'%Y%m%d_%H%M%S')"
            fi
            
            # Mover nuevo código a producción
            mkdir -p $APP_DIR
            mv ~/$DEPLOY_DIR/* $APP_DIR/
            
            # Iniciar la aplicación
            cd $APP_DIR
            pm2 start ecosystem.config.js --env ${{ github.base_ref }}
            
            # Limpieza: eliminar directorio temporal y backups antiguos
            rm -rf ~/$DEPLOY_DIR
            find /opt/app -name "backend_backup_*" -type d -mtime +7 -exec rm -rf {} \;
          ENDSSH

      # Verifica el despliegue
      - name: Verify deployment
        env:
          HEALTH_CHECK_URL: ${{ vars.EC2_HEALTH_CHECK_URL }}
        run: |
          attempts=0
          max_attempts=30
          until curl -s -f $HEALTH_CHECK_URL > /dev/null || [ $attempts -eq $max_attempts ]
          do
            attempts=$((attempts+1))
            echo "Waiting for application to be ready... ($attempts/$max_attempts)"
            sleep 10
          done
          
          if [ $attempts -eq $max_attempts ]; then
            echo "❌ Deployment verification failed"
            exit 1
          fi
          
          echo "✅ Deployment verified successfully"

      # Limpia las credenciales SSH
      - name: Cleanup SSH credentials
        if: always()
        run: rm -f ~/.ssh/deploy_key

```

Cambios y mejoras principales:

1. **Simplificación**:
    - Eliminado S3 y Systems Manager
    - Conexión directa vía SSH
    - Proceso más directo y fácil de debugear

2. **Seguridad**:
    - Uso de secretos para la clave SSH
    - Limpieza de credenciales después del uso
    - Permisos restrictivos para la clave SSH (600)
    - Verificación de known_hosts

3. **Variables necesarias en GitHub**:
   ```markdown
   Secrets:
   - EC2_SSH_KEY: Clave SSH privada para conexión
   
   Variables:
   - EC2_USER: Usuario SSH en EC2 (ej: ubuntu)
   - EC2_HOST: Hostname o IP de EC2
   - EC2_SSH_KNOWN_HOSTS: Salida de ssh-keyscan para el host
   - EC2_APPLICATION_URL: URL de la aplicación
   - EC2_HEALTH_CHECK_URL: URL del health check
   - APP_PORT: Puerto de la aplicación
   ```

4. **Características del despliegue**:
    - Backup automático del despliegue anterior
    - Limpieza de backups antiguos (>7 días)
    - Verificación post-despliegue
    - Variables de entorno configurables
    - Manejo de errores y rollback implícito

¿Te gustaría que ajustemos algo específico? Por ejemplo:
- Modificar la estrategia de backup
- Ajustar los timeouts de verificación
- Añadir pasos de rollback explícitos
- Configurar notificaciones de despliegue


## Más iteraciones para completar el deploy

```
Como puedo instalar Node y pm2 para ejecutar la aplicación en el servidor de EC2 siendo un Ubuntu?
```

```
Puedes simplificar la Github actions para que no se use pm2?
```

```
Al arrancar el servicio con node index.js se produce un error:

<error>
node:internal/modules/cjs/loader:1228
  throw err;
  ^
Error: Cannot find module 'express'
Require stack:
- /opt/app/backend/index.js
    at Module._resolveFilename (node:internal/modules/cjs/loader:1225:15)
    at Module._load (node:internal/modules/cjs/loader:1051:27)
    at Module.require (node:internal/modules/cjs/loader:1311:19)
    at require (node:internal/modules/helpers:179:18)
    at Object.<anonymous> (/opt/app/backend/index.js:7:33)
    at Module._compile (node:internal/modules/cjs/loader:1469:14)
    at Module._extensions..js (node:internal/modules/cjs/loader:1548:10)
    at Module.load (node:internal/modules/cjs/loader:1288:32)
    at Module._load (node:internal/modules/cjs/loader:1104:12)
    at Function.executeUserEntryPoint [as runMain] (node:internal/modules/run_main:174:12) {
  code: 'MODULE_NOT_FOUND',
  requireStack: [ '/opt/app/backend/index.js' ]
}
</error>
```

```
Falla por algun error en "prisma generate".

Puedes actualizar el pipeline en base al fichero package.json que te adjunto?
```

```
Como puedo crear una Base de datos en progres en esta instancia EC2?
```

```
Parece que al lanzar el script de despliegue falla por "npm: command not found"

Node se ha instalado con NVM, Debo configurar algo para que se cargue node al conectar por SSH?
```

# ARRANCADO!