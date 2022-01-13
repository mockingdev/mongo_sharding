# MongoDB sharding con docker

## Estructura de ficheros

```
├── build
│   ├── mongo
│   │   └── Dockerfile
│   ├── mongod
│   │   └── Dockerfile
│   └── mongos
│       └── Dockerfile
├── docker-compose.yml
├── mongod_luna.conf
├── mongod_config.conf
├── mongod_titan.conf
├── mongod_europa.conf
└── mongos.conf
```

## Construcción de Imágenes

Necesitamos 3 imágenes:

* el proceso `mongod` (que sostiene los `shards` y los `config server`),
* el proceso mongos (punto de entrada al cluster)
* el cliente `mongo` (que sirve para atar el cluster).

---

1. mongod Dockerfile:

```
FROM alpine:3.7
RUN apk add --no-cache mongodb && \
    rm /usr/bin/mongo /usr/bin/mongos /usr/bin/mongoperf && \
    install -d -o mongodb -g mongodb -m 0755 /srv/mongodb
USER mongodb
CMD ["/usr/bin/mongod", "--config", "/etc/mongod.conf"]
```

2. mongos Dockerfile:

```
FROM alpine:3.7
RUN apk add --no-cache mongodb && \
    rm /usr/bin/mongo /usr/bin/mongod /usr/bin/mongoperf
USER mongodb
CMD ["/usr/bin/mongos", "--config", "/etc/mongos.conf"]
```

3. mongo Dockerfile:

```
FROM alpine:3.7
RUN apk add --no-cache mongodb && \
    rm /usr/bin/mongod /usr/bin/mongos /usr/bin/mongoperf
USER mongodb
```

Construimos las imágenes:

```
$ docker build -t mongo-server build/mongod/
$ docker build -t mongo-proxy build/mongos/
$ docker build -t mongo-client build/mongo/
```

---

## Ejecución de los Procesos

Necesitamos un mínimo de 13 procesos:

* 1 mongos o más para poder utilizar e cluster de forma transparente
* 3 mongod en configuración de replica set para actuar como `config servers`
* 3 mongod en configuración de replica set para actuar como el `shard luna`
* 3 mongod en configuración de replica set para actuar como el `shard europa`
* 3 mongod en configuración de replica set para actuar como el `shard titan`

Para facilitar la ejecución de los procesos, vamos a utilizar `docker-compose`:

``` yaml
version: '3'
services:
  mongos01:
    image: mongo-proxy
    container_name: mongos01
    hostname: mongos01
    volumes:
      - ./mongos.conf:/etc/mongos.conf:ro
    restart: always
  config01:
    image: mongo-server
    container_name: config01
    hostname: config01
    volumes:
      - ./mongod_config.conf:/etc/mongod.conf:ro
    restart: always
  config02:
    image: mongo-server
    container_name: config02
    hostname: config02
    volumes:
      - ./mongod_config.conf:/etc/mongod.conf:ro
    restart: always
  config03:
    image: mongo-server
    container_name: config03
    hostname: config03
    volumes:
      - ./mongod_config.conf:/etc/mongod.conf:ro
    restart: always
  luna01:
    image: mongo-server
    container_name: luna01
    hostname: luna01
    volumes:
      - ./mongod_luna.conf:/etc/mongod.conf:ro
    restart: always
  luna02:
    image: mongo-server
    container_name: luna02
    hostname: luna02
    volumes:
      - ./mongod_luna.conf:/etc/mongod.conf:ro
    restart: always
  luna03:
    image: mongo-server
    container_name: luna03
    hostname: luna03
    volumes:
      - ./mongod_luna.conf:/etc/mongod.conf:ro
    restart: always
  europa01:
    image: mongo-server
    container_name: europa01
    hostname: europa01
    volumes:
      - ./mongod_europa.conf:/etc/mongod.conf:ro
    restart: always
  europa02:
    image: mongo-server
    container_name: europa02
    hostname: europa02
    volumes:
      - ./mongod_europa.conf:/etc/mongod.conf:ro
    restart: always
  europa03:
    image: mongo-server
    container_name: europa03
    hostname: europa03
    volumes:
      - ./mongod_europa.conf:/etc/mongod.conf:ro
    restart: always
  titan01:
    image: mongo-server
    container_name: titan01
    hostname: titan01
    volumes:
      - ./mongod_titan.conf:/etc/mongod.conf:ro
    restart: always
  titan02:
    image: mongo-server
    container_name: titan02
    hostname: titan02
    volumes:
      - ./mongod_titan.conf:/etc/mongod.conf:ro
    restart: always
  titan03:
    image: mongo-server
    container_name: titan03
    hostname: titan03
    volumes:
      - ./mongod_titan.conf:/etc/mongod.conf:ro
    restart: always
```

> Es importante que el `hostname` y el `container_name` sean el mismo; las replicas utilizan el `hostname` para su descubrimiento, pero el `container_name` al conectarse entre ellas.

Cada elemento dentro del cluster necesita un parámetro `replSetName` indicando el nombre de la replica set a la que pertenecen. Otro parámetro cambiante es el `clusterRole`, dependiendo si la replica set va a ejercer como `config server` o como `shard`. Los miembros del mismo replica set comparten configuración, así que solo necesitamos 4 distintas.

Fichero **mongod_config.conf**:

```
processManagement:
  fork: false

net:
  bindIp: 0.0.0.0
  port: 27019
  unixDomainSocket:
    enabled: false

storage:
  dbPath: /srv/mongodb
  engine: wiredTiger
  journal:
    enabled: true

replication:
  replSetName: config

sharding:
  clusterRole: configsvr
```

La configuración de los shards es prácticamente la misma; solo hace falta cambiar el `clusterRole` el `replSetName` y el puerto usado. Empezaremos exponiendo la configuración del primer shard:

Fichero **mongod_luna.conf**:

```
processManagement:
  fork: false

net:
  bindIp: 0.0.0.0
  port: 27018
  unixDomainSocket:
    enabled: false

storage:
  dbPath: /srv/mongodb
  engine: wiredTiger
  journal:
    enabled: true

replication:
  replSetName: luna

sharding:
  clusterRole: shardsvr
``` 

Los otros shards son prácticamente iguales, unicamente hay que cambiar el nombre de la `replica set`:

Fichero **mongod_europa.conf** y **mongod_titan.conf**:

```
...
replication:
  replSetName: europa
...

...
replication:
  replSetName: titan
...
```

Configuración del proceso `mongos`, fichero **mongos.conf**:

```
processManagement:
  fork: false

net:
  bindIp: 0.0.0.0
  port: 27017
  unixDomainSocket:
    enabled: false

sharding:
   configDB: config/config01:27019,config02:27019,config03:27019
```

Ejecutamos los procesos:

```
$ docker-compose up -d
Creating network "mongo_sharding_default" with the default driver
Creating luna03
Creating config03
Creating mongos01
Creating titan02
Creating europa03
Creating luna02
Creating luna01
Creating config01
Creating europa01
Creating titan03
Creating europa02
Creating config02
Creating titan01
```

## Configurando el Cluster

Para completar el cluster se precisa hacer dos cosas:

* Atar los replica sets que conformarán los `shards` y los `config server`
* Añadir los `shards` ya atados a través de un `mongos`

Para estas tareas levantado un cliente `mongo` mediante un contendor, con el requerimiento de añadirlo a la misma red que creo el `docker_compose.yml` para garantizar el poder usar los `container_name` en lugar de las direcciones IP de los contenedor.

```
$ docker run -ti --rm --net mongo_sharding_default mongo-client
```

> A partir de aquí todos los comandos se hacen en el shell de alpine linux.

Iniciamos los `config servers`:
```
$ mongo --host config01 --port 27019
...
> rs.initiate()
{
	"info2" : "no configuration specified. Using a default configuration for the set",
	"me" : "config01:27019",
	"ok" : 1
}

> rs.add("config02:27019")
{ "ok" : 1 }

> rs.add("config03:27019")
{ "ok" : 1 }

> exit
bye
```

Repetiremos el proceso para cada uno de los otros `shards`:

```
$ mongo --host luna01 --port 27018

> rs.initiate()
{
	"info2" : "no configuration specified. Using a default configuration for the set",
	"me" : "luna01:27018",
	"ok" : 1
}

> rs.add("luna02:27018")
{ "ok" : 1 }

> rs.addArb("luna03:27018")
{ "ok" : 1 }

> exit
bye
```

```
$ mongo --host europa01 --port 27018
...
> rs.initiate()
{
	"info2" : "no configuration specified. Using a default configuration for the set",
	"me" : "europa01:27018",
	"ok" : 1
}

> rs.add("europa02:27018")
{ "ok" : 1 }

> rs.addArb("europa03:27018")
{ "ok" : 1 }

> exit
bye
```

```
$ mongo --host titan01 --port 27018
...
> rs.initiate()
{
	"info2" : "no configuration specified. Using a default configuration for the set",
	"me" : "titan01:27018",
	"ok" : 1
}

> rs.add("titan02:27018")
{ "ok" : 1 }

> rs.addArb("titan03:27018")
{ "ok" : 1 }

> exit
bye
```

Ahora tenemos 4 replica sets, uno configurado como config server y apuntado por el proceso mongos, y otros 3 que serán los shards. Vamos a iniciar un mongo shell contra el proceso mongos, desde donde vamos a acabar las configuraciones.

```
$ mongo --host mongos01 --port 27017
...
mongos> 
```

De hecho, en este punto ya tenemos un cluster funcional, pero como no tiene shards, no hay donde guardar datos.

```
mongos> sh.status()
--- Sharding Status --- 
  sharding version: {
  	"_id" : 1,
  	"minCompatibleVersion" : 5,
  	"currentVersion" : 6,
  	"clusterId" : ObjectId("5b182ae62446c4f43cbab312")
  }
  shards:
  active mongoses:
        "3.4.10" : 1
  autosplit:
        Currently enabled: yes
  balancer:
        Currently enabled:  yes
        Currently running:  no
NaN
        Failed balancer rounds in last 5 attempts:  0
        Migration Results for the last 24 hours: 
                No recent migrations
  databases:

mongos> 
```

Para añadir `shards` solamente tenemos que utilizar el método `sh.addShard()` para especificar la replica set que va a actuar como `shard`; hay que añadir la replica set siguiendo la fórmula **rsName/server1:port,...,serverN:port**, aunque si especificamos uno solo nombre, basta.

```
mongos> sh.addShard("luna/luna01:27018")
{ "shardAdded" : "luna", "ok" : 1 }
mongos> 
```

> A pesar de haber dado solamente el nombre luna01, el resto de servidores ha sido descubierto por el cluster de forma automática; aún así, los árbitros no aparecen en el listado.

Vamos a repetir la fórmula para añadir los otros `shards`:

```
mongos> sh.addShard("titan/titan01:27018")
{ "shardAdded" : "titan", "ok" : 1 }
mongos> sh.addShard("europa/europa01:27018")
{ "shardAdded" : "europa", "ok" : 1 }
mongos> 
```

Y de esta forma, ya podemos ver el cluster acabado, con sus 3 shards añadidos sin problemas.

```
mongos> sh.status()
--- Sharding Status --- 
  sharding version: {
  	"_id" : 1,
  	"minCompatibleVersion" : 5,
  	"currentVersion" : 6,
  	"clusterId" : ObjectId("5b182ae62446c4f43cbab312")
  }
  shards:
        {  "_id" : "luna",  "host" : "luna/luna01:27018,luna02:27018",  "state" : 1 }
        {  "_id" : "europa",  "host" : "europa/europa01:27018,europa02:27018",  "state" : 1 }
        {  "_id" : "titan",  "host" : "titan/titan01:27018,titan02:27018",  "state" : 1 }
  active mongoses:
        "3.4.10" : 1
  autosplit:
        Currently enabled: yes
  balancer:
        Currently enabled:  yes
        Currently running:  no
NaN
        Failed balancer rounds in last 5 attempts:  0
        Migration Results for the last 24 hours: 
                No recent migrations
  databases:

mongos> 
```

Salimos del contenedor **mongos** para que el sistema lo pueda reciclar.

```
mongos> exit
bye

$ exit
```

Y con esto estamos listos para introducir nuestros datos.

