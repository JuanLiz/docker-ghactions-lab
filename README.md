# Taller: Implementación de canalizaciones CI/CD para imágenes de Docker con GitHub Actions y GitHub Packages

**Por Diego Lizarazo[^1]**

[^1]: Estudiante de Ingeniería de Sistemas y Computación de la Universidad de Cundinamarca. Microsoft Certified: Azure DevOps Engineer Expert y Azure Developer Associate. [GitHub](https://github.com/Juanliz) | [ORCID](https://orcid.org/0009-0002-2905-9092)

En este taller, aprenderemos a implementar un flujo de integración y entrega continua (CI/CD) para una aplicación sencilla en un contenedor Docker. Usaremos GitHub Actions para automatizar la construcción de la imagen de Docker, su publicación en GitHub Packages, y su posterior despliegue en un contenedor.

## Conceptos previos

### Docker

Docker es una plataforma que permite deplegar  aplicaciones aisladas en contenedores. Un contenedor es una unidad de software que contiene todo lo necesario para ejecutar una aplicación: las dependencias, el código, las bibliotecas y las configuraciones. Los contenedores son ligeros, portátiles, fáciles de compartir, lo que los hace ideales para implementar aplicaciones en entornos de desarrollo, pruebas y producción.

De igual manera, por estar aislados los contenedores no hay que preocuparse por las diferencias de configuración, ni que la ejecución de una aplicación afecte a otra, o al propio sistema operativo. Esta flexibilidad permite, por ejemplo, tener múltiples versiones de una misma aplicación, diferentes entornos de desarrollo, distintas versiones de lenguajes de programación, bibliotecas, etc. corriendo en un mismo sistema.

**A Docker siempre se le compara con una solución de virtualización tradicional**, como las máquinas virtuales. La principal diferencia es que Docker no virtualiza el hardware, sino que la abstracción es meramente hecha a nivel de software[^2]. Esto hace que los contenedores sean más ligeros y rápidos que las máquinas virtuales.

[^2]: Más información de la abstractación de Docker a nivel de software en [este artículo](https://coffeebytes.dev/es/container-de-docker-con-namespaces-y-cgroups/).

![Esquema: Contenedores vs Máquinas virtuales](https://www.cherryservers.com/v3/assets/blog/2022-12-20/01.jpg "Figura: Contenedores vs Máquinas virtuales")

#### Arquitecura de Docker: El motor y el registro de contenedores

Docker se compone de tres componentes principales,que para efecot prácticos los resumiremos en dos: El motor de Docker y el registro de contenedores.

- Los contenedores corren una instancia de una imagen dentro del propio motor de Docker. Es decir, el motor de Docker es el que se encarga de ejecutar los contenedores.

- **Una imagen es la base** que contiene todo lo necesario para ejecutar una aplicación, incluyendo el código, las bibliotecas, las dependencias, las variables de entorno, y las configuraciones. En otras palabras, es la plantilla para crear contenedores.

- Las imágenes se crean a partir del **Dockerfile**: Es un archivo, la receta para crear una imagen de Docker. En él se especifican todos los pasos que internamente Docker debe realizar para tener lista la imagen. Se crea en la raíz del proyecto que se quiere contenerizar.

- Las imágenes se almacenan en un registro de contenedores **(Container registry)**, que es un repositorio de imágenes, de donde se pueden descargar y subir imágenes. Docker Hub es el registro de contenedores en línea oficial de Docker, pero también se pueden usar otros o incluso crear uno propio.

- Por defecto, el motor de Docker tiene de forma local un registro de contenedores, en el cual guarda las imágenes que se van creando.

![Esquema: Funcionamiento de registro de contenedores](https://www.tutorialspoint.com/docker/images/docker_hub_1.jpg "Figura: Funcionamiento de registro de contenedores")

### GitHub Actions y GitHub Packages

GitHub Actions es un servicio de automatización de flujos de trabajo que permite construir, probar y desplegar código directamente desde GitHub. Se pueden crear flujos de trabajo personalizados, o usar plantillas predefinidas para automatizar tareas comunes.

![GitHub Actions](https://github.com/images/modules/site/actions/fp24/hero.webp "Figura: GitHub Actions")

GitHub Packages es un servicio de registro de paquetes de GitHub. Permite a los desarrolladores publicar y compartir paquetes de código, imágenes de contenedores, y otros artefactos de software. Los paquetes se pueden usar en proyectos de GitHub, o en cualquier otro lugar donde se necesiten.

Luego, GitHub Actions se encargará de automatizar la construcción de la imagen y su publicación en GitHub Packages. También se puede automatizar el despliegue de la imagen en un contenedor, pero en este taller lo haremos manualmente. Para este caso, como vamos a cargar la imagen de Docker, GitHub Packages lo cargará en su registro de contenedores (GitHub Container Registry).

![GitHub actions + GitHub Packages](https://repository-images.githubusercontent.com/216605017/627c7780-57db-11ea-990b-17c6ffdff523 "Figura: GitHub actions + GitHub Packages")

## Partes del presente taller

Entendiendo estos conceptos, en este taller aprenderemos a:

1. **Integrar Docker en un proyecto existente**, creando un Dockerfile para la generación de una imagen de Docker. Para este caso, usaremos una aplicación de Node.js sencilla, la cual es la plantilla inicial de un proyecto de Next.js. Si se desea, se puede usar cualquier otro proyecto, de cualquier otro lenguaje, plataforma, entorno, etc.
2. **Automatizar la construcción de la imagen y su publicación en el registro de contenedores GitHub Packages**, usando GitHub Actions mediante la configuración de un flujo de trabajo en un archivo de configuración.
3. **Desplegar la imagen en un contenedor**, obteniendo la imagen desde GitHub Packages y ejecutándola en el entorno local.

## Requisitos previos

Aparte de tener nociones básicas de Git y el uso de la línea de comandos, para este taller se requiere lo siguiente:

- Tener una cuenta en GitHub.
- Tener instalado Docker Engine en la máquina local. Puede descargarse desde [este enlace](https://docs.docker.com/engine/install/). No es necesario instalar Docker Desktop, ya que **no vamos a usar una interfaz gráfica.**
- Tener un repositorio en GitHub con el código de la aplicación, en caso de no tenerlo, se puede usar la aplicación de ejemplo de este taller (carpeta sample-app)

## Parte 1: Integración de Docker en un proyecto existente

### Generación del Dockerfile

En Docker se usa la programación declarativa: Se especifica qué es lo que se quiere, y Docker se encarga de determinar cómo lo hará.

El Dockerfile es la receta para crear una imagen de Docker. En él se especifican todos los pasos que internamente Docker debe realizar para tener lista la imagen.

#### Docker Hub

La creación de una imagen de Docker empieza desde una imagen base. Podemos encontrar las principales imágenes base en [Docker Hub](https://hub.docker.com/). Hay de todo tipo: De sistemas operativos, de frameworks, de lenguajes de programación, de bases de datos, etc.

Para este caso buscaremos una imagen de Node.js que nos sirva como base para nuestra aplicación. Debería entonces coincidir con la versión del proyecto.

#### Ejemplo de Dockerfile

El siguiente es un ejemplo de Dockerfile para una aplicación de Node.js. Se puede cambiar según se requiera. Al final se explicarán de manera simple los comandos usados.

El Dockerfile deberá crearse también en la carpeta raíz del proyecto.

```dockerfile
# Imagen base. Se pone el alias base para usarla en tareas aparte
FROM node:lts-alpine AS base

# Se toma la imagen base para compilar la aplicación
FROM base AS builder
RUN apk add --no-cache libc6-compat
WORKDIR /app
# Copiar los archivos de dependencias de la aplicación a la imagen
COPY package.json package-lock.json* ./
# Instalar las dependencias
RUN npm ci
COPY . .
# Compilar la aplicación con npm
RUN npm run build

# Se vuelva a tomar la imagen base, ahora para la ejecución de la aplicación
FROM base AS runner
WORKDIR /app
# Definir variables de entorno
ENV NODE_ENV production
# Añadir grupo y usuario al sistema
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

# Ajustar permisos de los archivos
RUN mkdir .next
RUN chown nextjs:nodejs .next
# Copiar los archivos de la imagen de compilación a la imagen final
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

# Definiendo el usuario que ejecutará la aplicación
USER nextjs
# Puerto que se va a exponer para acceder a la aplicación
EXPOSE 3000
ENV PORT 3000

# Comando usado para iniciar la aplicación cuando se ejecute el contenedor
CMD HOSTNAME="0.0.0.0" node server.js
```

> [!IMPORTANT]
> El siguiente ejemplo no es una solución única. Cada aplicación tiene sus propias necesidades y requerimientos. Se recomienda leer la documentación oficial de Docker[^3] para entender mejor cada comando.

[^3]: [Referencia de Dockerfile](https://docs.docker.com/reference/dockerfile/)

#### Comandos usados en el Dockerfile

En el ejemplo dado se usaron los comandos básicos de un Dockerfile. A continuación, una breve explicación de cada uno:

- `FROM`: Indica la imagen base que se usará para construir la imagen. En este caso, se usó `node:lts-alpine` como base para la imagen. Posteriormente, se indicó con `AS` un alias para la imagen, que se usó en los entornos de compilación y ejecución.
- `RUN`: Ejecuta comandos en la imagen. En este caso, se usó para instalar las dependencias de la aplicación.
- `WORKDIR`: Establece el directorio de trabajo dentro del contenedor. En este caso, se estableció `/app` como directorio de trabajo.
- `COPY`: Copia archivos de la máquina local al contenedor. Su sintaxis es `COPY <from> <to>`. En este caso, se copiaron los archivos de la aplicación al directorio de trabajo interno de la imagen.
- `ENV`: Establece variables de entorno en la imagen. En este caso, se establecieron las variables de Node.js y el puerto de la aplicación.
- `EXPOSE`: Define la exposición de un puerto para el contenedor. En este caso, se expuso el puerto 3000. Según la aplicación, se puede cambiar el puerto o abrir varios puertos.
- `USER`: Define el usuario que ejecutará la aplicación en el contenedor. En este caso, se definió el usuario `nextjs`.
- `CMD`: Define el comando que se ejecutará cuando el contenedor se inicie. **No es lo mismo que** `RUN`, ya que `CMD` se ejecuta cuando el contenedor se inicia, mientras que `RUN` se ejecuta cuando se construye la imagen.

> [!TIP]
> Puedes ayudarte de asistentes de IA (Copilot, ChatGPT, etc.) para crear tu Dockerfile. También puedes usar plantillas predefinidas. Se encuentran fácil en las páginas de documentación de los lenguajes y frameworks.

#### Creación de la imagen

Para crear la imagen se ejecuta el comando `docker build`, el nombre de la imagen, y la ubicación del Dockerfile. Para este caso, el punto al final indica que el Dockerfile está en la carpeta actual.

```bash
docker build -t <nombre-de-la-imagen> .
```

#### Verificar que la imagen se creó

```bash
docker images
```

Con lo anterior, ya tenemos la imagen de Docker creada y lista para ser usada.

## Parte 2: Automatización de la construcción de la imagen y su publicación en GitHub Packages con GitHub Actions

### Partes de los flujos de trabajo

Según la documentación oficial de GitHub Actions[^4], un flujo de trabajo se compone de varias partes:

- **Eventos:** Son los desencadenantes que inician el flujo de trabajo. Suele ser un evento de GitHub, como un push, un pull request, la aprtura de una incidencia (issue), etc.
- **Trabajos:** Son las tareas que se ejecutan en paralelo en el flujo de trabajo. Cada trabajo se ejecuta en un entorno de ejecución (runner) y puede contener varios pasos.
- **Acciones:** Son las tareas individuales que se ejecutan en un trabajo. Como usualmente se repiten, ya existen acciones predefinidas que se pueden usar y conseguir desde el Marketplace de GitHub Actions o crearlas personalizadas. Para este caso, usaremos acciones predefinidas.
- **Entornos de ejecución (runners):** Son las máquinas virtuales o contenedores que ejecutan los trabajos. GitHub provee máquinas virtuales con Windows, Ubuntu Linux y macOS. Cada flujo de trabajo se ejecuta en un runner nuevo.

[^4]: [Documentación oficial de GitHub Actions](https://docs.github.com/en/actions/about-github-actions/understanding-github-actions)

### Archivo de configuración de GitHub Actions

Para automatizar la construcción de la imagen y su publicación en GitHub Packages, se debe crear un archivo de configuración de GitHub Actions. Este archivo se debe crear en la carpeta `.github/workflows` del repositorio. Si no existen las carpetas, se deben crear también.

El archivo se escribe en formato YAML y así como Docker, se basa en la programación declarativa.

#### Configuración para el presente taller

Para cumplir con los objetivos del taller, se debe crear un flujo de trabajo que realice los siguientes pasos:

1. **Definir el evento que desencadenará el flujo de trabajo.** En este caso, será un push a la rama principal del repositorio.
2. **Definir las variables de entorno necesarias.** En este caso, se necesitará la URL del registro de contenedores de GitHub Packages (`ghcr.io`) y el nombre de la imagen.
3. **Definir el trabajo que se ejecutará.** En este caso, solo se ejecutará un trabajo, que se encargará de construir la imagen y publicarla en GitHub Packages. El trabajo contendrá los siguientes pasos:

   1. **Definir el entorno de ejecución.** En este caso, se usará un runner de Ubuntu Linux.
   2. **Definir los permisos para la autenticación**. GitHub crea automáticamente un token de autenticación para el flujo de trabajo, pero debemos especificar los permisos necesarios para que pueda leer el contenido del repositorio y publicar la imagen en GitHub Packages.
   3. **Sincronizar el código del repositorio en le entorno de ejecución,** usando la acción predefinida `actions/checkout@v4`.
   4. **Iniciar sesión en el registro de contenedores,** usando la acción predefinda que provee Docker: `docker/login-action@v3`. Tendremos que especificar qué registro de contenedor usaremos, en este caso el de GitHub Packages (`ghcr.io`), junto con las respectivas credenciales.
   5. **Extraer los metadatos de la imagen,** usando la acción predefinida `docker/metadata-action@v6`. Esta acción extrae los metadatos de la imagen, como las etiquetas, la arquitectura, el sistema operativo, etc.
   6. **Construir la imagen y publicarla en GitHub Packages,** usando la acción predefinida `docker/build-push-action@v3`. En esta acción, se especifica la ruta del Dockerfile, el nombre de la imagen, la etiqueta, y el registro de contenedores (GitHub Packages).

#### Ejemplo de archivo de configuración

Dados los pasos anteriores, el archivo de configuración de GitHub Actions debería verse así:

```yaml
# Los nombres se pueden cambiar. Se recomienda que sean descriptivos
name: Construir y publicar imagen de Docker

# Definición del evento que desencadenará el flujo de trabajo. Push en la rama principal
on:
  push:
    branches: [ main ]

# Definicón de variables de entorno
env:
  # URL del registro de contenedores 
  REGISTRY: ghcr.io
  # El nombre de la imagen está definido como el nombre del repositorio
  IMAGE_NAME: ${{ github.repository }}

# Definición de los trabajos que se ejecutarán
jobs:
  build-push-registry:
    name: Construir y publicar imagen de Docker al registro de contenedores de GitHub Packages
    # Definicón del entorno de ejecución (runner)
    runs-on: ubuntu-latest

    # Definición de los permisos necesarios para el flujo de trabajo
    permissions:
      packages: write
      contents: read
      id-token: write

    # Definicón de los pasos que se ejecutarán en el trabajo
    steps:
      - name: Sincronizar el código
        uses: actions/checkout@v4

      - name: Iniciar sesión en el registro de contenedores
        uses: docker/login-action@v3
        with:
          # Tomar URL de las variables de entorno
          registry: ${{ env.REGISTRY }}
          # El nombre de usuario es el mismo que el de GitHub
          username: ${{ github.actor }}
          # La contraseña es el token generado por GitHub
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extraer metadatos de la imagen
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}

      - name: Construir y publicar imagen en GitHub Packages
        id: push
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

Para incluir este ejemplo en el repositorio, se debe crear la carpeta `.github/workflows` y dentro de ella, crear un archivo con el nombre que se desee, pero con la extensión `.yml`. Por ejemplo, `docker.yml`.

#### Ejecución del flujo de trabajo

Para ejecutar el flujo de trabajo, se debe hacer un push a la rama principal del repositorio. GitHub Actions detectará el evento y ejecutará el flujo de trabajo.

Para verificar que el flujo de trabajo se ejecutó correctamente, se puede ir a la pestaña "Actions" del repositorio en GitHub.

![Ejecución de GitHub Actions](https://docs.github.com/assets/images/help/repository/actions-tab-actions.png "Figura: Ejecución de GitHub Actions")

## Parte 3: Deslpiegue de la imagen en un contenedor

Recordemos que el contenedor es la instancia de la imagen. Es decir, es la imagen ejecutándose. Para ejecutar la imagen, se usa el comando `docker run` y se especifica el nombre de la imagen con los parámetros adicionales.

### Especificar el puerto

Para que el contenedor pueda ser accedido desde el exterior, se debe especificar el puerto en el que la aplicación se ejecutará. Para eso, se usa el parámetro `-p <puerto-externo>:<puerto-interno>`.

```bash
docker run -p 3000:3000 <nombre-de-la-imagen>
```

### Ejecutarse en segundo plano

Para que el contenedor se ejecute en segundo plano, se usa el parámetro `-d`.

```bash
docker run -d -p 3000:3000 <nombre-de-la-imagen>
```

### Especificar el nombre del contenedor

Por defecto, Docker asigna un nombre aleatorio al contenedor. Para especificar el nombre, se usa el parámetro `--name <nombre-del-contenedor>`.

```bash
docker run -d -p 3000:3000 --name <nombre-del-contenedor> <nombre-de-la-imagen>
```



### Consideraciones: Base de datos local

En escenarios de desarollo y producción, las aplicaciones se conectan a bases de datos mediante strings de conexión. Pero debido a que, para este caso estaremos usando una base de datos local, y Docker usa su propio espacio de red, **no podremos acceder a la base de datos desde el contenedor.**

Si usamos `localhost`, el contenedor intentará acceder a su propia base de datos, que no existe, porque está intentando acceder a su red interna:

![Red interna de Docker](https://res.cloudinary.com/practicaldev/image/fetch/s--Md8BNSlM--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3fj5w1jrpqqpu7gv2kww.jpg)

Para solucionar esto, tenemos varias opciones, como cambiar las configuraciones de conexión de la aplicación, agregar una red, usar docker-compose, etc. Por practicidad, en este caso usaremos el parámetro `--network host` para que el contenedor use la misma red que el host (nuestra máquina).

![Alt text](https://res.cloudinary.com/practicaldev/image/fetch/s--5H94WFmE--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/bclomlpxgr8rqgqzm2ta.jpg)

```bash
docker run -d -p 3000:3000 --network host \
--name <nombre-del-contenedor> <nombre-de-la-imagen>
```

### Verificar que el contenedor se está ejecutando

```bash
docker ps
```

## Conclusiones

- En este laboratorio aprendimos los comandos básicos para realizar un despliegue de una aplicación en un contenedor Docker.

- Quedan muchos temas por explorar, como el uso de volúmenes, redes, gestión de secretos, docker-compose, etc. Animo a que sean investigados y practicados para familiarizarse con ellos.

- Docker se usa actualmente en la mayoría de escenarios de desarrollo y producción, por lo que es importante conocerlo y saber usarlo.

## Referencias

- [Documentación oficial de Docker](https://docs.docker.com/)
- [Docker Hub](https://hub.docker.com/)
- [Referencia de Dockerfile](https://docs.docker.com/engine/reference/builder/)