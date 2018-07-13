# PARCIAL 2 SISTEMAS DISTRIBUIDOS #

***1.Consigne los comandos de linux necesarios para el aprovisionamiento de los servicios solicitados. 
En este punto no debe incluir archivos tipo Dockerfile solo se requiere que usted identifique los comandos o 
acciones que debe automatizar***

***CONTENEDOR ELASTICSEARCH***
```bash
docker run -d --name=elastic elasticsearch
```

***CONTENEDOR KIBANA***
```bash
docker run -d --name=kibana --link elastic:elasticsearch -p 5601:5601 kibana
```

***CONTENEDOR FLUENTD***

Para que fluentd pueda comunicarse con elasticsearch es necesario instalar un plugin. Para ello se utiliza un archivo Dockerfile que 
instala este plugin sobre la imagen oficial de fluentd

```dockerfile
FROM fluent/fluentd:v0.12
RUN ["gem", "install", "fluent-plugin-elasticsearch", "--no-rdoc", "--no-ri", "--version", "1.9.5"]
```

Para crear la imagen con el plugin:
```bash
docker build -t fluentd-elastic . 
```

Para crear el contenedor:
```bash
docker run --name=fluentd  --link elastic:elasticsearch -v /fluentd/conf:/fluentd/etc -e FLUENTD_CONF=fluent.conf fluentd-elastic
```

***CONTENEDOR CON SERVICIO WEB***

El servicio web utilizado es httpd. 
```bash
docker run -d -p 80:80 --name=apache --log-driver=fluentd --log-opt fluentd-address=172.17.0.4:24224 --log-opt tag=httpd.access httpd
```
Para que este servicio se pueda comunicar con fluentd hay que especificar la ip del contenedor fluentd. En este caso, la ip era 172.17.0.4.
Tambien se podria exponer el servicio del contenedor fluentd para que fuera accesible desde localhost. 
Este metodo es el utilizado en el docker-compose.


***2.Escriba los archivos Dockerfile para cada uno de los servicios solicitados junto con los archivos fuente necesarios. 
Tenga en cuenta consultar buenas prácticas para la elaboración de archivos Dockerfile***

Unicamente se utiliza un archivo Dockerfile para la creacion de la imagen de fluentd con el plugin de elasticsearch

```dockerfile
FROM fluent/fluentd:v0.12
RUN ["gem", "install", "fluent-plugin-elasticsearch", "--no-rdoc", "--no-ri", "--version", "1.9.5"]
```



***3.Escriba el archivo docker-compose.yml necesario para el despliegue de la infraestructura***

```yml
version: '2'
services:
  web:
    image: httpd
    ports:
      - "80:80"
    links:
      - fluentd
    logging:
      driver: "fluentd"
      options:
        fluentd-address: localhost:24224
        tag: httpd.access

  fluentd:
    build: ./fluentd
    volumes:
      - ./fluentd/conf:/fluentd/etc
    links:
      - "elasticsearch"
    ports:
      - "24224:24224"
      - "24224:24224/udp"

  elasticsearch:
    image: elasticsearch

  kibana:
    image: kibana
    links:
      - "elasticsearch"
    ports:
      - "5601:5601"
```


***4.Incluya evidencias que muestran el funcionamiento de lo solicitado***

Para poner en funcionamiento se debe utilizar el siguiente comando:
```bash
docker-compose up -d
```

Primero se debe acceder al servicio de kibana expuesto en el puerto 5601 y crear un index para la visualizacion de los logs provenientes
de fluentd.

![index](https://user-images.githubusercontent.com/17281733/32993611-9f8f3526-cd28-11e7-9c9f-3ed4313e8a15.png)

Una vez hecho esto, se pueden visualizar los logs:


![img1](https://user-images.githubusercontent.com/17281733/32993965-39946d26-cd2e-11e7-8637-71518bb12afb.png)


![img2](https://user-images.githubusercontent.com/17281733/32993970-41d65b0c-cd2e-11e7-80f6-c8453005be03.png)



***5.Documente algunos de los problemas encontrados y las acciones efectuadas para su solución al aprovisionar 
la infraestructura y aplicaciones***

Este ejercicio fue realizado empleando docker for mac. Cuando se quizo crear el contenedor de fluentd por medio de comandos se obtenia
el siguiente error:

ERROR:

![error](https://user-images.githubusercontent.com/17281733/32994020-48f64c52-cd2f-11e7-8bd8-e58e92df8a92.png)

CAUSA:

En docker for mac se puede compartir archivos que esten dentro de los siguientes directorios siempre y cuando se haga explicita la
ruta completa:


![causa](https://user-images.githubusercontent.com/17281733/32994026-7a25d2ca-cd2f-11e7-8b42-c61aa449c0d6.png)

SOLUCION:
```bash
docker run --name=fluentd --link elastic:elasticsearch -v /Users/Carlos/Downloads/prueba/fluentd/conf/:/fluentd/etc -e FLUENTD_CONF=fluent.conf fluentd-elastic
```
