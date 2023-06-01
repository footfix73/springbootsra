#  Cómo configurar un Jenkins Pipeline en OpenShift

  

OpenShift se integra con Jenkins para ayudar a crear rápidamente **pipelines de CI/CD** utilizando tecnologías que ya conocemos.

Pero OpenShift es mucho más que Kubernetes. Es una plataforma completa que nos ayuda a pasar del código al contenedor. Los pipelines de CI/CD están integrados directamente.

  
#  Cómo configurar pipelines de Jenkins en OpenShift

Configuráramos un pipeline de Jenkins en OpenShift. Cuando terminemos, podremos hacer push a un repositorio de Git y el pipeline de Jenkins construirá una imagen de contenedor y la desplegará en OpenShift.
 
Lo que necesitamos:

- Conocimientos básicos de Git
- Experiencia básica con Jenkins
- Una cuenta de GitHub

##  Paso 1: Crear un entorno de OpenShift y Jenkins

Primero, configuraremos un entorno de OpenShift. La forma más sencilla de hacerlo es creando un entorno de desarrollo sandbox. Podemos obtenerlos desde [aquí](https://developers.redhat.com/developer-sandbox/get-started).

Una vez que tu sandbox esta configurado, estaremos listos para iniciar Jenkins. OpenShift viene con una imagen incorporada para ello. Para nuestros propósitos, crearemos un servidor Jenkins.

Bajo la pestaña de **Developer**, haz clic en el botón **"Add"** y selecciona "**All Services**".

En la pestaña de búsqueda, escribimos **"jenkins ephemeral"**. Hacemos clic en **"Instantiate Template"** y seleccionamos **"Create"**.

Esto creará un contenedor de Jenkins y una URL para acceder a él.

Hacemos clic en la pestaña **"Topology"** para ver el progreso del servidor Jenkins. Seleccionamos el despliegue de Jenkins. Cuando termine de implementarse, veremos un pod en estado **"running"**.

A continuación, verificáramos si Jenkins se desplegó correctamente. En la misma página de la aplicación, nos desplazamos hasta la parte inferior y hacemos clic en la URL de la ruta.

Si vemos la página de inicio de sesión de Jenkins, ¡hemos creado correctamente un servidor Jenkins!

## Paso 2: Crear un Pipeline de Jenkins 
Creamos un pipeline de Jenkins que construirá un contenedor a partir de nuestro código fuente. Para este tutorial, utilizaremos un proyecto de ejemplo en Java. Es un proyecto sencillo de **"Hola mundo"** que se ejecuta en Java EE.

En primer lugar, hacemos un **fork** del repositorio de código fuente de ejemplo [aquí](https://github.com/footfix73/springbootsra.git). Luego, clonamos el repositorio en local y cambiamos a la rama de pipelines.

````
git clone <your_url> 
git checkout openshift-jenkins-pipeline 
cd springbootsra
````

Abrimos el archivo Jenkins. Este es un pipeline de plantilla que modificaremos para construir, crear una imagen de contenedor y desplegar en OpenShift.

Agrega lo siguiente al bloque de pasos (steps) debajo de la etapa de construcción (build).

```
sh 'mvn clean package'
```

Este paso compila el código fuente, lo empaqueta en un archivo JAR y lo colocará en la carpeta "target" en nuestro servidor de compilación.

A continuación, tomamos el artefacto y construimos una imagen de contenedor. La forma más sencilla de hacerlo es utilizando [OpenShift Source-to-Image (S2I)](https://docs.openshift.com/container-platform/4.7/openshift_images/using_images/using-s21-images.html?ref=springbootsra.org).  Este proceso combina el código fuente con una imagen de compilador para crear un contenedor ejecutable.

S2I está integrado en OpenShift. Pero para acceder a él, necesitamos utilizar el complemento OpenShift Jenkins Client. **Afortunadamente, este complemento ya está incluido en las imágenes de contenedor de Jenkins en OpenShift.**

Primero, volvemos a la consola de OpenShift y obtenemos el nombre de nuestro proyecto.

Luego, en la etapa **"Create Container Image"** del Jenkinsfile, agrega lo siguiente entre las etiquetas de script.  Nos aseguramos de reemplazar el nombre del proyecto con el tuyo.
````
openshift.withCluster() { 
  openshift.withProject("<tu_nombre_proyecto>") {
  
    def buildConfigExists = openshift.selector("bc", "springbootsra").exists() 
    
    if(!buildConfigExists){ 
      openshift.newBuild("--name=springbootsra", "--docker-image=registry.redhat.io/jboss-eap-7/eap74-openjdk8-openshift-rhel7", "--binary") 
    } 
    
    openshift.selector("bc", "springbootsra").startBuild("--from-file=target/simple-servlet-0.0.1-SNAPSHOT.war", "--follow") } }
````

Utilizando el complemento de OpenShift, el código realiza algunas acciones:

- **Verificar si existe una configuración de construcción (build config)**. No queremos crear una en cada ejecución del pipeline. 

- **Crear una configuración de construcción (build config)**. Las BuildConfigs describen cómo se construyen las imágenes de contenedor en OpenShift. 

- **Iniciar una nueva construcción (build)**. Source to Image puede utilizar código fuente o un binario. Aquí se tomará nuestro archivo WAR y la imagen del compilador para construir un contenedor ejecutable. 

Finalmente, necesitamos desplegar nuestro contenedor en OpenShift. Bajo el bloque de pasos (steps) en la etapa "Desplegar" (Deploy), añadimos lo siguiente entre las etiquetas de script. Asegúrate de reemplazar el nombre del proyecto con el tuyo.

```
openshift.withCluster() { 
  openshift.withProject("<tu_nombre_proyecto") { 
    def deployment = openshift.selector("dc", "springbootsra") 
    
    if(!deployment.exists()){ 
      openshift.newApp('springbootsra', "--as-deployment-config").narrow('svc').expose() 
    } 
    
    timeout(5) { 
      openshift.selector("dc", "springbootsra").related('pods').untilEach(1) { 
        return (it.object().status.phase == "Running") 
      } 
    } 
  } 
}
```

Al igual que con nuestra configuración de construcción (build config), verificamos si existe un despliegue existente. Si no existe, crearemos una nueva aplicación. `oc new-app` crea todos los recursos necesarios para desplegar tu contenedor. Una vez que se haya completado, obtendremos una URL para acceder a la aplicación.

La última línea nos ayuda a monitorear el estado de este despliegue. Solo queremos marcar esta etapa como completada cuando nuestros contenedores estén en ejecución.

Guarda los cambios y haz push a tu repositorio.
```
git add . && git commit -m “Update jenkins pipeline” && git push
```
## Paso 3: Crear la Configuración de Construcción 
Ahora que tenemos nuestro pipeline de Jenkins, necesitamos indicarle a OpenShift que lo utilice para nuestras construcciones. Para ello, utilizaremos una BuildConfig.

Selecciona **"Builds"** en la consola y luego haz clic en **"Create BuildConfig"**.

Cambia el nombre de tu configuración de construcción a `mi-pipeline`. A continuación, actualizamos la URI de Git y la referencia (ref) para que coincida con tu repositorio de Git. Aquí es donde buscará tu archivo de pipeline de Jenkins.
```
uri: <tu-repo-url> 
ref: openshift-jenkins-pipeline
```
Reemplaza la sección de **"strategy"** de construcción (build strategy) para que se vea de la siguiente manera:
```
strategy:
    jenkinsPipelineStrategy:
      jenkinsfilePath: Jenkinsfile
```
Esto indica a nuestra configuración de construcción que utilizaremos un pipeline de Jenkins para construir y desplegar nuestra imagen de contenedor. Haz clic en "Create" para crear una nueva configuración de construcción.

Finalmente, agregamos un **webhook** a la configuración de construcción. Abre la **ventana de comandos de OpenShift** representado por la siguiente imagen **>_**.

Cuando se cargue, ingresa lo siguiente:

```
oc set triggers bc/mi-pipeline --from-github
```

Cierra la terminal y regresa a la configuración de construcción. Cuando actualices la página, deberías ver un webhook en la parte inferior. Haz clic en **"Copy URL with secret"**.

Ve a tu cuenta de GitHub, abre el repositorio y haz clic en **"Settings"**.

Abrimos la sección de **webhooks** y crea un nuevo webhook con las siguientes especificaciones:
```
Payload Url: <tu-webhook-url-copiada>
Content Type: application/json
```
Haz clic en **"Create Webhook"**.

Si todo está configurado correctamente, nuestro pipeline de Jenkins se ejecutará cada vez que hagamos push a nuestro repositorio. ¡Probemoslo!

En tu equipo, vamos a la carpeta que contiene nuestro servlet.
```
cd src/main/java/org/springbootsra/servlet
```
Abre **SimpleServlet.java** y cambia el texto "Hola Mundo" por "Hola <tu-nombre>" o lo que se quiera poner. Guarda los cambios y haz push a tu repositorio.

Regresa a tu configuración de construcción (**mi-pipeline**) y haz clic en "**Builds**". Selecciona la construcción recién iniciada y deberías ver las etapas del pipeline en ejecución.

Cuando haya terminado, regresamos a la **"Topology"**. Haz clic en la aplicación llamada **springbootsra** y encuentra la URL de ruta en la parte inferior. Navega a <route_url>/simple-servlet-0.0.1-SNAPSHOT/SimpleServlet.

¡Deberíamos ver la pantalla con tu mensaje actualizado!

## Mejores prácticas 
OpenShift cuenta con imágenes de Jenkins completamente compatibles, pero OpenShift Pipelines ofrece una experiencia nativa en la nube para CI/CD. Con OpenShift Pipelines, puedes ejecutar pipelines de forma nativa en Kubernetes sin tener que gestionar un servidor de construcción. 

## Resumen 
Kubernetes es excelente para gestionar contenedores, pero no ofrece mucho para los desarrolladores. Sin embargo, OpenShift te ayuda a pasar del código al contenedor. Hay soporte integrado para pipelines de Jenkins, así como un complemento de cliente de OpenShift para Jenkins.

Durante este tutorial, creamos un pipeline de Jenkins para construir y desplegar nuestra imagen de contenedor a partir de código fuente de Java. Luego creamos una configuración de construcción en OpenShift para ejecutar el pipeline desde un webhook.
