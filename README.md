# Todomvc

Este repo está construido con [Gradle](https://gradle.org/), [JUnit 5](http://junit.org/junit5/), [Selenium](http://www.seleniumhq.org/) y el código está realizado en Java.

Al ejecutar la tarea de test de Gradle, Selenium levantará un Chrome usando el Chromedriver (que viene incluido en este proyecto) y realizará los siguientes test sobre la página web de [todomvc.com](http://todomvc.com/examples/vanillajs/):

* El título es el correcto
* Crear una tarea
* Borrar una tarea
* Editar una tarea
* Filtrar las tareas
* Completar una tarea
* Borrar las tareas completadas

![Todomvc Tests](./todomvc-tests.gif)

1. Introducción
Con esta guía conseguiremos montar un entorno de integración continua reproducible y escalable con Jenkins y Selenium Grid. Haremos uso de los nuevos Pipelines de Jenkins para definir nuestra build y haremos que se construya nuestro proyecto y mediante variables de entorno que pasaremos a Gradle poder ejecutar los tests funcionales tanto local como remotamente.

Antes de nada se recomiendan tener conocimientos de Java, Gradle, Docker, Jenkins y sistemas.

2. Requisitos
Java JDK 1.8.0_144
Docker version 17.09.0-ce (Instalar con Docker for Mac o Docker for Windows)
Chrome v61
3. Entorno
MacBook Pro (Retina, 15-inch, Mid 2014)
MacOS High Sierra 10.13
iTerm 2 + ZSH
IntelliJ IDEA 2017.2.5
VSCode 1.17.1
4. Clonar repositorio de todomvc y ejecutar los tests
He preparado un repositorio que hace uso de tests funcionales con Selenium. Para que podáis usarlo en el entorno de integración continua tendréis que o clonarlo y subirlo a vuestro propio repo o hacer un fork. Sin olvidaros claro de darle ⭐️.

Este repo está construido con Gradle, JUnit 5 y Selenium. Hacemos uso de tecnologías como Headless Chrome para que nuestros tests se ejecuten de forma más rápida.

Los test se han hecho sobre la página de todomvc y se realizan los siguientes tests:

El título es el correcto
Crear una tarea
Borrar una tarea
Editar una tarea
Filtrar las tareas
Completar una tarea
Borrar las tareas completadas
Para ver en vuestra máquina la ejecución de los tests, una vez tengáis el proyecto en local ejecutar el siguiente comando desde la línea de comandos en el directorio del proyecto:


./gradlew clean build -P debug=true
1
./gradlew clean build -P debug=true
Con debug=true hacemos que se muestre en primer plano y que uso nuestra instalación de Chrome.

Como hacemos uso de Gradle Wrapper no hará falta que tengamos descargado Gradle en nuestra máquina.

5. Docker
En este apartado montaremos un Jenkins y un Selenium Grid con docker-compose. Realizaremos unos pequeños ajustes en la imagen de Jenkins y construiremos la nuestra propia con un Dockerfile.

Comenzamos creando un directorio sobre el que trabajaremos:


take ~/todomvc-jenkins
1
take ~/todomvc-jenkins
En este directorio tendremos los ficheros de Dockerfile, docker-compose.yml (que crearemos en un paso posterior) y el volumen de datos de Jenkins.

Creamos un fichero Dockerfile:


touch Dockerfile
1
touch Dockerfile
En este fichero lo que haremos será construir nuestro entorno de CI con Jenkins. Hacemos uso de un Dockerfile ya que queremos lograr dos cosas:

Instalar y configurar xvfb en nuestro servidor
Instalar los plugins que vamos a usar, en este caso Blueocean
Con nuestro editor favorito (VSCode, cómo no) incluir lo siguiente en el fichero Dockerfile:


FROM jenkins/jenkins:2.84

USER root

ENV DISPLAY=:0
RUN echo $DISPLAY

# Set up Xvfb as in Linux servers there is no display by default
RUN apt-get update
RUN apt-get install -y xvfb
RUN xvfb :0 -screen 0 1024x768x24 &> xvfb.log &

# Preinstall needed plugins
RUN /usr/local/bin/install-plugins.sh blueocean

USER jenkins
FROM jenkins/jenkins:2.84
 
USER root
 
ENV DISPLAY=:0
RUN echo $DISPLAY
 
# Set up Xvfb as in Linux servers there is no display by default
RUN apt-get update
RUN apt-get install -y xvfb
RUN xvfb :0 -screen 0 1024x768x24 &> xvfb.log &
 
# Preinstall needed plugins
RUN /usr/local/bin/install-plugins.sh blueocean
 
USER jenkins
¿Por qué necesitamos xvfb? Xvfb es necesario ya que en los servidores de Linux no hay ningún display por defecto, y el no tenerlo haría que fallasen todos nuestros tests.

6, Editar /etc/hosts
Para no tener que lidiar con IPs para comunicar nuestros contenedores vamos a editar nuestro fichero hosts para mapear las siguientes urls a localhost:

todomvc-jenkins
selenium-hub
Abrimos y editamos el fichero /etc/hosts:


nano /etc/hosts
1
nano /etc/hosts
Nano etc hosts

Introducimos en el mapeo de 127.0.0.1 lo siguiente:


selenium-hub todomvc-jenkins
selenium-hub todomvc-jenkins
Nano etc hosts

Guardamos ctrl + O y salimos ctrl + x. Si queréis usar un editor de terminal más completo que Nano pero no tan complicado como Vim os recomiendo Micro.

7. Docker compose
Con docker-compose podemos gestionar la creación y orquestación de contenedores desde un único punto.

Crear un fichero docker-compose.yml e incluir lo siguiente:


# Versión de Docker Compose
version: '3.3'
services:
  jenkins:
    # Con el build decimos que haga uso de la imagen que se construirá con el Dockerfile de este directorio
    build: .
    # Exponemos el puerto 49001 en nuestra máquina local para poder ver la interfaz de Jenkins
    ports:
      - "49001:8080"
    # Incluimos un volumen de datos en el directorio actual bajo la carpeta jenkins
    volumes:
      - ./jenkins:/var/jenkins_home
  # Contenedor de Selenium Grid
  selenium-hub:
    image: selenium/hub:3.6.0
    # Exponemos el puerto para comprobar la configuración de Selenium Hub
    ports:
      - "49002:4444"
  # Contenedor de Chrome
  chrome:
    image: selenium/node-chrome:3.6.0
    # Inlcuimos links para conectar el contendor de Chrome al Hub
    links:
      - selenium-hub:hub
    # Variables de entorno necesarias
    environment:
      - HUB_PORT_4444_TCP_ADDR=hub
      - HUB_PORT_4444_TCP_PORT=4444

# Versión de Docker Compose
version: '3.3'
services:
  jenkins:
    # Con el build decimos que haga uso de la imagen que se construirá con el Dockerfile de este directorio
    build: .
    # Exponemos el puerto 49001 en nuestra máquina local para poder ver la interfaz de Jenkins
    ports:
      - "49001:8080"
    # Incluimos un volumen de datos en el directorio actual bajo la carpeta jenkins
    volumes:
      - ./jenkins:/var/jenkins_home
  # Contenedor de Selenium Grid
  selenium-hub:
    image: selenium/hub:3.6.0
    # Exponemos el puerto para comprobar la configuración de Selenium Hub
    ports:
      - "49002:4444"
  # Contenedor de Chrome
  chrome:
    image: selenium/node-chrome:3.6.0
    # Inlcuimos links para conectar el contendor de Chrome al Hub
    links:
      - selenium-hub:hub
    # Variables de entorno necesarias
    environment:
      - HUB_PORT_4444_TCP_ADDR=hub
      - HUB_PORT_4444_TCP_PORT=4444
Para levantar los contenedores usar el siguiente comando:


docker-compose up
1
docker-compose up
A continuación veríamos en consola como se van levantando los distintos contenedores. Pura magia:

Docker compose

8. Configuración inicial de Jenkins
Ir a todomvc-jenkins:49001:

Desbloquear Jenkins

La contraseña inicial debería aparecer en consola:

Contraseña inicial Jenkins en terminal

Si por alguna razón no la vemos, nos tenemos que meter dentro del contenedor de Jenkins con docker exec.

Primero miramos los contenedores que tenemos activos:


docker ps --format "{{.Names}}"
1
docker ps --format "{{.Names}}"
Docker ps images

He usado –format dado que si no no se ve bien los nombres de los contenedores. Si queréis ver toda la información de los contenedores usar docker ps.

En este caso el nombre de mi contenedor es todomvcjenkins_jenkins_1, con lo que introduzco el siguiente comando:


docker exec -it todomvcjenkins_jenkins_1 bash
1
docker exec -it todomvcjenkins_jenkins_1 bash
Y estamos dentro:

Docker exec Jenkins

Ahora mostramos la contraseña inicial con el siguiente comando:


cat /var/jenkins_home/secrets/initialAdminPassword
1
cat /var/jenkins_home/secrets/initialAdminPassword
Deberíamos ver algo parecido a esto:

Contraseña inicial Jenkins

Copiamos la contraseña y la pegamos en la pantalla de desbloquear Jenkins.

8.1. Instalar plugins

Dado que en nuestro Dockerfile hemos incluido la siguiente línea: RUN /usr/local/bin/install-plugins.sh blueocean Jenkins ya se ha encargado de instalar el único plugin que vamos a necesitar, con lo que damos seleccionar plugins a instalar y seleccionamos ninguno. Le damos a siguiente.

8.2. Crear usuario inicial

Para esta guía le daremos a continuar como usuario administrador.

El usuario por tanto será admin y la contraseña será la contraseña inicial.

8.3. BlueOcean

BlueOcean es la nueva propuesta del equipo de Jenkins para actualizar la imagen de Jenkins. Está hecho con React y últimas tecnologías web. Si te interesa React aquí tienes un tutorial escrito por mí.

Vamos a pasar a configurar nuestro proyecto de Github con BlueOcean para que se integren.

Pinchar en BlueOcean:

Abir BlueOcean

Ahora veremos la interfaz de BlueOcean:

Interfaz BlueOcean

Daremos click en Create a new Pipeline:

Crear nuevo pipeline con servidor

En esta guía haremos uso de Github como servicio para gestionar nuestro repositorio de Git.

Por tanto hacemos click en Github y nos pedirá un token de acceso:

Token de acceso Jenkins

Hacemos click en el link de Create an access token here y este nos llevará a la página de creación de tokens de Github:

Token de acceso en Github

Le damos a generar sin desmarcar ninguna opción:

Generar token de acceso en Github

Y seguidamente copiamos el token que nos aparece en pantalla y lo pegamos en BlueOcean:

Pegar token de acceso en BlueOcean

Una vez conectado nos apareceran las distintas cuentas de las que disponemos y nuestros repositorios.

Cuentas:

Cuentas github

Hacemos click en nuestra cuenta (en mi caso le daré click en cesalberca)

Y seguidamente seleccionamos nuestro repo todomvc del cuál hemos fork o hemos clonado y subido a nuestra cuenta de Github:

Repo github

Y por último le damos a New pipeline:

Nuevo pipeline

Con este último paso Jenkins se conectará a nuestro repo de Github, lo clonará, y lo inspeccionará para ver si este cuenta con un fichero Jenkinsfile. Si lo tiene ejecutará los pasos definidos en nuestro Jenkinsfile, que si os acordáis tenía la siguiente pinta:


#!/usr/bin/env groovy

pipeline {
  agent any

  stages {
    stage('Test') {
      steps {
        sh './gradlew clean build -P env=prod'
      }
    }
    stage('Clean up') {
      steps {
        deleteDir()
      }
    }
  }
}

#!/usr/bin/env groovy
 
pipeline {
  agent any
 
  stages {
    stage('Test') {
      steps {
        sh './gradlew clean build -P env=prod'
      }
    }
    stage('Clean up') {
      steps {
        deleteDir()
      }
    }
  }
}
Con lo que lanzará los tests en modo producción con Selenium-grid. Y dado que hemos montado nuestros contenedores con docker-compose éstos se pueden ver y comunicar entre sí, ya que están en la misma red.

Es importante abstraer la configuración, de tal forma que podamos cambiarla fácilmente. Es por esto por lo que tengo especificado en mi clase Config.java lo siguiente:


package com.autentia.training.selenium.todomvc.config;

public final class Config {

    public static final String HUB_URL = System.getProperty("hub.url", "http://selenium-hub:4444/wd/hub");
    public static final String BASE_URL = System.getProperty("base.url", "http://todomvc.com/examples/vanillajs/");
    public static final String ENV = System.getProperty("env", "dev");
    public static final boolean DEBUG = Boolean.valueOf(System.getProperty("debug", "false"));

    private Config() {
    }

}

package com.autentia.training.selenium.todomvc.config;
 
public final class Config {
 
    public static final String HUB_URL = System.getProperty("hub.url", "http://selenium-hub:4444/wd/hub");
    public static final String BASE_URL = System.getProperty("base.url", "http://todomvc.com/examples/vanillajs/");
    public static final String ENV = System.getProperty("env", "dev");
    public static final boolean DEBUG = Boolean.valueOf(System.getProperty("debug", "false"));
 
    private Config() {
    }
 
}
De esta forma podré sobreescribir las propiedades una vez lance los tests. Dado que los tests son lanzados desde Gradle, le tendremos que pasar las propiedades a Gradle, y Gradle las tendrá que pasar a la JVM.

Lo hago de la siguiente forma desde build.gradle:


// Pass from the Gradle JVM to Java's JVM the system properties
junitPlatformTest {
    systemProperties project.properties.subMap(["env", "debug", "browser", "hub.url", "base.url"])
}

// Pass from the Gradle JVM to Java's JVM the system properties
junitPlatformTest {
    systemProperties project.properties.subMap(["env", "debug", "browser", "hub.url", "base.url"])
}
Con lo que cuando ejecutemos Gradle con el Wrapper de Gradle podremos sobreescribir las variables de la siguiente forma:


./gradlew clean build -P hub.url=http://other-selenium-hub/ -P base.url=http://test.todomvc.com
1
./gradlew clean build -P hub.url=http://other-selenium-hub/ -P base.url=http://test.todomvc.com
Ahora veremos que Jenkins instrospecciona el repositorio, mira el Jenkinsfile, y ejecuta las fases con sus pasos. Si todo ha ido bien veremos que se lanza una primera build automáticamente y esta debería pasar:

Jenkins Success
9. Extras
Una vez llegados a este punto no habrá quien nos pare. Os dejo un par de ideas para hacer de vuestro entorno de CI el mejor de todos:

Hacer que Selenium registre dos navegadores más
Ejecutar los tests en distintos navegadores en paralelo con los Pipeline
Lanzar los tests automáticamente cada vez que se haga un merge
Notificar a Github si la build se ha hecho correctamente para poder aprobar el merge
Notificar por correo/Slack/SMS en caso de que la build falle
Pasar los tests automáticamente todos los días al finalizar el día
Mostrar la cobertura de tests
Usar docker-compose para autoescalar el número de navegadores
Ejecutar los tests en distintas plataformas (Windows, Mac y Linux) con Selenium Grid
