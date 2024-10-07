<style>

*, pre, code, kbd, samp, tt {
    font-family: "CaskaydiaCove Nerd Font", "Cascadia Code", monospace;
}

h1, h2, h3{
    font-family: "Lato Black", sans-serif;
}

h4, h5, h6 {
    font-family: "Lato Semibold", sans-serif;
}

p, em, strong, a, ul, ol, li {
    font-family: "Lato", sans-serif;
}

</style>

# Taller: Implementación de canalizaciones CI/CD para imágenes de Docker con GitHub Actions y GitHub Packages

**Por Diego Lizarazo[^1]**

[^1]: footnote text is displayed at the end of the page.

En este taller, aprenderemos a implementar un flujo de integración y entrega continua (CI/CD) para una aplicación sencilla en un contenedor Docker. Usaremos GitHub Actions para automatizar la construcción de la imagen de Docker, su publicación en GitHub Packages, y su posterior despliegue en un contenedor.

## Conceptos previos

### Docker

Docker es una plataforma que permite deplegar  aplicaciones aisladas en contenedores. Un contenedor es una unidad de software que contiene todo lo necesario para ejecutar una aplicación: las dependencias, el código, las bibliotecas y las configuraciones. Los contenedores son ligeros, portátiles, fáciles de compartir, lo que los hace ideales para implementar aplicaciones en entornos de desarrollo, pruebas y producción.

De igual manera, por estar aislados los contenedores no hay que preocuparse por las diferencias de configuración, ni que la ejecución de una aplicación afecte a otra, o al propio sistema operativo. Esta flexibilidad permite, por ejemplo, tener múltiples versiones de una misma aplicación, diferentes entornos de desarrollo, distintas versiones de lenguajes de programación, bibliotecas, etc. corriendo en un mismo sistema.

**A Docker siempre se le compara con una solución de virtualización tradicional**, como las máquinas virtuales. La principal diferencia es que Docker no virtualiza el hardware, sino que la abstracción es meramente hecha a nivel de software[^2]. Esto hace que los contenedores sean más ligeros y rápidos que las máquinas virtuales.

[^2]: Más información de la abstractación de software de Docker en [este artículo](https://coffeebytes.dev/es/container-de-docker-con-namespaces-y-cgroups/).

![Esquema: Contenedores vs Máquinas virtuales](https://www.cherryservers.com/v3/assets/blog/2022-12-20/01.jpg "Figura: Contenedores vs Máquinas virtuales")

### Arquitecura de Docker: El motor y el registro de contenedores

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
2. **Cargar la imagen a un registro de contenedores**, en este caso GitHub Packages.
3. **Automatizar la construcción de la imagen y su publicación en GitHub Packages**, usando GitHub Actions mediante la configuración de un flujo de trabajo en un archivo de configuración.
4. **Desplegar la imagen en un contenedor**, obteniendo la imagen desde GitHub Packages y ejecutándola en el entorno local.

## Requisitos previos

Aparte de tener nociones básicas de Git y el uso de la línea de comandos, para este taller se requiere lo siguiente:

- Tener una cuenta en GitHub.
- Tener instalado Docker Engine en la máquina local. Puede descargarse desde [este enlace](https://docs.docker.com/engine/install/). No es necesario instalar Docker Desktop, ya que no vamos a usar una interfaz gráfica.
- Tener un repositorio en GitHub con el código de la aplicación, en caso de no tenerlo, se puede usar la aplicación de ejemplo de este taller (carpeta sample-app)


## Parte 1: Integración de Docker en un proyecto existente

xd

### Generación del Dockerfile

En Docker se usa la programación declarativa: Se especifica qué es lo que se quiere, y Docker se encarga de determinar cómo lo hará.

El Dockerfile es la receta para crear una imagen de Docker. En él se especifican todos los pasos que internamente Docker debe realizar para tener lista la imagen.

#### Docker Hub

Para la mayoría de casos, la creación de una imagen de Docker empieza desde una imagen base (una plantilla). El hub oficial de imágenes de Docker se encuentra en [Docker Hub](https://hub.docker.com/). Hay de todo tipo: De sistemas operativos, de frameworks, de lenguajes de programación, de bases de datos, etc.

Para este caso buscaremos una imagen de Node.js que nos sirva como base para nuestra aplicación. Debería entonces coincidir con la versión del proyecto.

#### Ejemplo de Dockerfile

El siguiente es un ejemplo de Dockerfile para una aplicación de Node.js. Se puede cambiar según se requiera.

```dockerfile
# La imagen base. Generalmente de Docker Hub. Cambiar por la versión a usar
FROM node:14

# Establece el directorio de trabajo dentro del contenedor
WORKDIR /app

# Copia el archivo package.json y package-lock.json (dependencias)
COPY package*.json ./

# Instala las dependencias de la aplicación
RUN npm install

# Copia el resto de los archivos de la aplicación (COPY <from> <to>)
COPY . .

# Exponer el puerto en el que la aplicación se ejecutará
EXPOSE 3000

# Comando para ejecutar la aplicación
ENTRYPOINT ["node", "app.js"]
```

#### Creación de la imagen

Crear la imagen se hace con el comando `docker build`, el nombre de la imagen, y la ubicación del Dockerfile. Para este caso, el punto al final indica que el Dockerfile está en la carpeta actual.

```bash
docker build -t <nombre-de-la-imagen> .
```

#### Verificar que la imagen se creó

```bash
docker images
```

## Ejecutando la imagen en un contenedor

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