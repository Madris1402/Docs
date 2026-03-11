---
tags:
---
### Configuración Inicial
 ***Configurar Quartz localmente***

1. Abre tu terminal y clona el repositorio oficial de Quartz:
```powershell
git clone https://github.com/jackyzha0/quartz.git
cd quartz
```
    
2. Instalar NodeJS si no se tiene instalado
```powershell
winget.exe uninstall --id "OpenJS.NodeJS" --exact --source winget --accept-source-agreements --disable-interactivity --version "25.8.0" --silent
```
2. Instala las dependencias necesarias con npm:
```powershell
npm i
```
    
3. Inicializa la configuración de Quartz. El asistente te hará algunas preguntas básicas (puedes dejar las opciones por defecto):
```powershell
npx quartz create
```
    
***Conectar tu Bóveda (Vault) de Obsidian***
Quartz busca tus notas en la carpeta llamada `content`. Tienes un par de opciones:

- **Opción A (Copiar):** Copia las notas que quieras publicar desde tu bóveda de Obsidian y pégalas dentro de la carpeta `content` del directorio de Quartz. (La que uso yo)

- **Opción B (Sincronizar - Recomendado):** Si tienes tu bóveda en otra ubicación, puedes crear un enlace simbólico (symlink) para que Quartz lea directamente desde tu bóveda sin duplicar archivos. También puedes simplemente abrir la carpeta `quartz` directamente como tu nueva bóveda en Obsidian.

Para probar que todo funciona bien localmente, ejecuta:

```powershell
npx quartz build --serve
```

Esto levantará un servidor local (usualmente en `http://localhost:8080`) donde podrás ver cómo lucen tus notas.

### Subir a GitHub

1. Ve a GitHub y crea un **nuevo repositorio vacío** (por ejemplo, `mis-notas`).
    
2. En tu terminal, dentro de la carpeta de Quartz, elimina el origen del repositorio original de Quartz y enlaza el tuyo:
```powershell
git remote remove origin
git remote add origin https://github.com/TU_USUARIO/mis-notas.git
```
    
3. Haz el primer commit y súbelo a tu repositorio:
```powershell
git add .
git commit -m "Inicializando Quartz con mis notas de Obsidian"
git push -u origin v4
```
- _(Nota: Quartz usa la rama `v4` por defecto, asegúrate de hacer push a esa rama o a `main` si la renombraste)._

### Configurar GitHub Pages con GitHub Actions

Quartz 4 ya viene con un archivo de configuración para automatizar el despliegue.

1. En tu repositorio de GitHub, ve a la pestaña **Settings** (Configuración).
2. En el menú lateral izquierdo, haz clic en **Pages**.
3. En la sección **Build and deployment**, busca la opción **Source** y cámbiala de "Deploy from a branch" a **"GitHub Actions"**.
4. Al hacer esto, GitHub Actions detectará automáticamente el archivo de flujo de trabajo de Quartz que subiste en tu repositorio.
    

Comenzará a compilar tu sitio estático y, en un par de minutos, te mostrará el enlace de tu nueva página (usualmente `https://tu-usuario.github.io/mis-notas/`).

#### Si no apareciera el flujo oficial de Quartz en Git.
Esto significa que GitHub no encuentra el archivo de instrucciones para construir tu página.

***Revisar la configuración de Pages en GitHub***
A veces, si esto no está activado, GitHub simplemente ignora los flujos de trabajo de despliegue.

1. En tu repositorio en GitHub, ve a **Settings** (Configuración).
2. En el menú de la izquierda, haz clic en **Pages**.
3. En **Build and deployment**, asegúrate de que el **Source** (Origen) esté configurado estrictamente como **GitHub Actions**.

###  Crear el archivo de flujo de trabajo

Es muy probable que en tu carpeta local falte el archivo que le dice a GitHub qué hacer.

1. En tu computadora, ve a la carpeta de tu proyecto (donde tienes Quartz).
2. Entra a la carpeta oculta `.github`, luego a la carpeta `workflows`. (Si no existen, créalas).
3. Dentro de `workflows`, crea un archivo de texto llamado `deploy.yml`.
4. Ábrelo con cualquier editor de código (o bloc de notas) y pega exactamente este código

```yaml
name: Deploy Quartz site to GitHub Pages

on:
  push:
    branches:
      - v4

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0 # Fetch all history for git info
      - uses: actions/setup-node@v4
        with:
          node-version: 22
      - name: Install Dependencies
        run: npm ci
      - name: Build Quartz
        run: npx quartz build
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: public

  deploy:
    needs: build
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

***Subir los cambios***
Guarda el archivo y, en tu terminal, sube este nuevo archivo a tu repositorio de GitHub usando el comando de sincronización de Quartz:

```powershell
npx quartz sync
```

Una vez que subas esto, si regresas a la pestaña **Actions** en GitHub (la misma de tu captura), deberías ver aparecer un nuevo flujo llamado **"Deploy Quartz site to GitHub Pages"** con un circulito amarillo girando, lo que indicará que por fin está construyendo tu página web.

### Cómo actualizar los permisos del entorno:

1. En tu repositorio de GitHub, ve a la pestaña **Settings** (Configuración).
2. En el menú lateral izquierdo, busca y haz clic en **Environments** (Entornos).
3. En la lista, verás uno llamado **`github-pages`**. Haz clic sobre el nombre para editar sus reglas.
4. Baja hasta encontrar la sección **Deployment branches and tags** (Ramas y etiquetas de despliegue).
5. En el menú desplegable (que probablemente diga "Selected branches"), cámbialo a **"All branches"** (Todas las ramas).
    - _Alternativa:_ Si quieres ser más estricto con la seguridad, déjalo en "Selected branches", haz clic en "Add deployment branch rule", escribe `v4` y guárdalo.
6. Asegúrate de guardar los cambios si la página te lo pide.

***Reintentar el despliegue***

Una vez que actualizaste este permiso, no necesitas volver a sincronizar ni subir nada desde tu terminal. Podemos simplemente reintentar el proceso que falló:

1. Ve a la pestaña **Actions**.
2. Haz clic en el flujo de trabajo que falló (el que tiene la "X" roja).
3. En la parte superior derecha de esa pantalla, haz clic en el botón **Re-run jobs** (Volver a ejecutar trabajos) y selecciona **Re-run failed jobs**.

---

**Flujo de trabajo a futuro:** Cada vez que escribas o modifiques una nota en Obsidian, solo tendrás que hacer lo siguiente en tu terminal:

```powershell
npx quartz sync
```

Este comando empaqueta el `git add`, `git commit` y `git push` en un solo paso. Al subir los cambios, GitHub Actions actualizará tu página web automáticamente.

¿Te gustaría que profundicemos en cómo personalizar el diseño (colores, tipografía) de tu nuevo sitio, o prefieres ayuda configurando el enlace simbólico de tus directorios?

### Compatibilidad con Archivos html ya terminados
Tanto Obsidian como Quartz tienen la capacidad nativa de leer y renderizar etiquetas HTML crudas **siempre y cuando estén dentro de un archivo Markdown**.

1. Ve a donde tienes tu reporte de PSR+ (`archivo.html`).
2. Simplemente **cámbiale la extensión** de `.html` a `.md` (por ejemplo, `archivo_psr.md`).
3. Mételo de vuelta en la carpeta de tu bóveda.

Al hacer esto, engañas al sistema:

- **En Obsidian:** El motor detecta que es una nota `.md`, por lo que te permite enlazarla usando los clásicos `[[archivo_psr]]`. Al abrir la nota, Obsidian leerá las etiquetas HTML de PSR+ y te renderizará la tabla o el texto visualmente sin problemas.

- **En Quartz:** El compilador verá un archivo `.md` normal. Al procesarlo, el plugin interno (`rehype-raw`) respetará todo el código HTML que viene adentro y lo inyectará directamente en la página final.