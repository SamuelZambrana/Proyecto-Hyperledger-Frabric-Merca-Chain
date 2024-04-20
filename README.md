# Solución práctica

## Ejercicio 1
**Enunciado:** _Despliega la red. Parece que existen algunos contenedores que no se levantan correctamente. Identifícalos y resuelve los problemas asociados._

**Solución:**

En primer lugar, algunos de los scripts de esta red no son ejecutables y obtenemos "Permiso denegado" o errores similares. Podemos ver los permisos de los archivos con "ll". Así mismo, podemos solucionarlo ejecutando "sudo chmod +x" a los scripts:
```bash
cd hf-certification-practica 
sudo chmod +x install-fabric.sh
cd merca-chain
sudo chmod +x network.sh
cd scripts
sudo chmod +x *
```

Ejecutamos el script install-fabric.sh para instalar los binarios:
```bash
install-fabric.sh b
```

Ya estamos listos para desplegar la red:
```bash
cd merca-chain
./network.sh up -ca
```

Si ejecutamos "docker ps -a" podremos identificar de un vistazo los contenedores caídos (aquellos cuyo estado es "Exited"). Son el **peer0.org1.example.com** y el **orderer.example.com**.
> 7b5c778f324a   hyperledger/fabric-peer:latest      "peer node start"        26 seconds ago   Exited (1) 24 seconds ago      peer0.org1.example.com
>
> fff714aa98a3   hyperledger/fabric-orderer:latest   "orderer"                26 seconds ago   Exited (2) 23 seconds ago      orderer.example.com

Podemos ver los logs de los contenedores caídos usando el comando "docker logs <CONTAINER_ID>". En mi caso:
```bash
docker logs 7b5c778f324a
docker logs fff714aa98a3
```

En el caso de **peer0.org1.example.com** se trata de un error que tiene que ver con su clave tls:
> 2024-04-19 15:00:42.936 UTC 0005 FATA [nodeCmd] serve -> Error loading secure config for peer (error loading TLS key (open /etc/hyperledger/fabric/tls/server.key: no such file or directory))

En el caso de **orderer.example.com** el error es bastante similar
> 2024-04-19 15:00:43.537 UTC 000e PANI [orderer.common.server] Main -> failed to start admin server: open /var/hyperledger/orderer/tls/servre.crt: no such file or directory
>
> panic: failed to start admin server: open /var/hyperledger/orderer/tls/servre.crt: no such file or directory

Este error puede deberse a varios motivos:
- Los certificados no se han generado correctamente.
- Las rutas de los certificados no están bien configuradas en el fichero docker-compose de la red (_compose-test-net.yaml_).

Tras una breve indagación en la estructura de directorios del repositorio (los certificados se generan en el directorio _/organizations_) podemos ver que los certificados están en su lugar. 

Sin embargo para el caso del orderer, las rutas especificadas en el fichero _compose-test-net.yaml_ contienen un typo y no se corresponden con las rutas reales de los certificados (_/tls/**servre**.crt_ -> _/tls/server.crt_).

En el caso del peer, el error es otro typo y se encuentra en el montaje de volúmenes (_/etc/**hypre**ledger/fabric_ -> _/etc/hyperledger/fabric_).

Tras la corrección de estos pequeños errores podemos reiniciar la red y todos los contenedores aparecerán corriendo correctamente.

```bash
./network.sh down
./network.sh up -ca
docker ps -a
```

> Un **typo** en informática consiste en un bug provocado provocado por una errata o un error tipográfico involuntario al teclear.

![Image 1](./images/image1.PNG?raw=true "Contenedores corriendo correctamente")

## Ejercicio 2
**Enunciado:** _Despliega el chaincode “merca-chaincode”. Aunque el proceso de ciclo de vida del chaincode es satisfactorio, parece que este no se despliega correctamente. Identifica el/los problemas y resuélvelos. Documenta este proceso de investigación y resolución._

**Solución:**

Ahora que la red se despliega al completo, podemos crear el canal y desplegar el chaincode:
```bash
./network.sh createChannel
./network.sh deployCC -ccn merca-chaincode -ccl javascript -ccv 1.0.0 -ccp ../merca-chaincode
```

Al acabar estos comandos podemos comprobar los contenedores de los contratos inteligentes con "docker ps -a", lo cual nos arrojará que el contrato inteligente no está corriendo para ambos peers (org1 y org2):
> 31ffd99a28ac   dev-peer0.org1.example.com-merca-chaincode_1.0.0-2fe5761cf062628f09fa31a0e7c29059db695faee2cfa5797153416ff970bdb3-66eb2055c9999f2e317d5d5115c6bd531c1189d02d2645dbf6f93359e4e25794   "docker-entrypoint.s…"   2 minutes ago   Exited (1) 2 minutes ago                                                                                                                                     dev-peer0.org1.example.com-merca-chaincode_1.0.0-2fe5761cf062628f09fa31a0e7c29059db695faee2cfa5797153416ff970bdb3
> 50f21ec394b5   dev-peer0.org2.example.com-merca-chaincode_1.0.0-2fe5761cf062628f09fa31a0e7c29059db695faee2cfa5797153416ff970bdb3-71dd532d0117e94771c67c951510c00ffff2b805f11fe658695e775658682142   "docker-entrypoint.s…"   2 minutes ago   Exited (1) 2 minutes ago                                                                                                                                     dev-peer0.org2.example.com-merca-chaincode_1.0.0-2fe5761cf062628f09fa31a0e7c29059db695faee2cfa5797153416ff970bdb3

Podemos acceder a los logs de los contenedores para analizar qué ha podido salir mal:
```bash
docker logs 50f21ec394b5
docker logs 31ffd99a28ac
```

El error en cuestión dice así:
> [...]
> /usr/local/src/lib/assetTransfer.js:92
>     async ReadAsset(ctx, id) {
>           ^^^^^^^^^
> SyntaxError: Unexpected identifier
> [...]

Si vamos a la línea 92 del fichero _/usr/local/src/lib/assetTransfer.js_ tal y como nos indica el error, veremos sin mucho esfuerzo que falta cerrar un corchete "}". El hacer uso de un entorno de desarrollo como VS Code nos puede hacer ganar tiempo en este punto. Arreglado esto, tenemos dos opciones para desplegar el chaincode correctamente:
1. Tirar abajo la red completamente y volver a hacer una instalación limpia.
2. Crear una actualización del contrato inteligente que sirva de parche para este error:
```bash
./network.sh deployCC -ccn merca-chaincode -ccl javascript -ccv 1.0.1 -ccs 2 -ccp ../merca-chaincode
```

Por último constatamos con "docker ps" que los contenedores correspondientes al chaincode están corriendo. Si hay muchos contenedores podemos hacerle un favor a nuestra vista filtrando con el comando "grep":
```bash
docker ps | grep merca-chaincode
```

## Ejercicio 3
**Enunciado:** _Desde Merca-link nos comentan que los logs de los peers no proporcionan la información deseada. Modifica el logging de la red para que se ajuste a los siguientes requerimientos:_
- _Los logs de los orderers deben estar en formato JSON._
- _Los logs de los peers deben de mostrar la fecha y la hora al final._
- _Dado que estamos en fase de pruebas, los logs de los peers deben de tener un nivel de logging de DEBUG por defecto. Además, los logs asociados al logger “gossip” tendrán un nivel por defecto de WARNING y los del logger de “chaincode” de INFO para no sobrecargar demasiado los logs._

**Solución:**

Estas configuraciones se pueden realizar de distintas maneras:
1. Edición del fichero de configuración del nodo (_core.yaml_ para peers y _orderer.yaml_ para orderers). En concreto, habría que modificar la siguiente sección:
```yaml
# Logging section for the chaincode container
    logging:
      # Default level for all loggers within the chaincode container
      level:  info
      # Override default level for the 'shim' logger
      shim:   warning
      # Format for the chaincode container logs
      format: '%{color}%{time:2006-01-02 15:04:05.000 MST} [%{module}] %{shortfunc} -> %{level:.4s} %{id:03x}%{color:reset} %{message}'
```
2. Mediante variables de entorno en el fichero docker-compose de la red (_compose/compose-test-net.yaml_). Yo opté por esta opción pues es más portable.

Los cambios se pueden ver en dicho fichero y son los siguientes:
- _Los logs de los orderers deben estar en formato JSON._ 
```yaml
    [...]
    environment:
      - FABRIC_LOGGING_SPEC=INFO
      - FABRIC_LOGGING_FORMAT=json # Añadimos esta línea en las variables del orderer
      - ORDERER_GENERAL_LISTENADDRESS=0.0.0.0
    [...]
```
- _Los logs de los peers deben de mostrar la fecha y la hora al final._
- _Dado que estamos en fase de pruebas, los logs de los peers deben de tener un nivel de logging de DEBUG por defecto. Además, los logs asociados al logger “gossip” tendrán un nivel por defecto de WARNING y los del logger de “chaincode” de INFO para no sobrecargar demasiado los logs._
```yaml
    [...]
    # Hacemos los cambios para ambos peers:
    environment:
      - FABRIC_CFG_PATH=/etc/hyperledger/peercfg
      - FABRIC_LOGGING_SPEC=debug:gossip=warning:chaincode=info # Con esta línea aplicamos los cambios en los niveles de severidad
      - FABRIC_LOGGING_FORMAT='%{color} [%{module}] %{shortfunc} -> %{level:.4s} %{id:03x}%{color:reset} %{message} %{time:2006-01-02 15:04:05.000 MST}' # Con esta línea aplicamos los cambios en la posición de la fecha y hora (al final)
      - CORE_PEER_TLS_ENABLED=true
    [...]
```

Reiniciamos la red y verificamos la aplicación de los cambios realizados con el comando "docker logs <CONTANIER_ID>" Para los peers y el orderer. 

![Image 2](./images/image2.PNG?raw=true "Logs de peer en DEBUG por defecto")

![Image 3](./images/image2.PNG?raw=true "Logs de orderer en formato JSON")

## Ejercicio 4
**Enunciado:** _Los técnicos de Merca-link nos muestran su preocupación acerca de las claves criptográficas, que se almacenan en texto plano dentro de los directorios de la red. Como una primera medida de securización de las claves y de las operaciones criptográficas sugieren el acoplamiento de un HSM a las CAs, utilizándose softhsm2 para esta fase de pruebas y configurando un token diferente para cada CA._

**Solución:**

### Instalación y configuración de SoftHSM2
Comenzamos instalando softhsm2:
```bash
sudo apt install softhsm2
```

Por defecto la configuración de SOFTHSM2 está en /etc/softhsm2.conf. Exportamos la siguiente variable de entorno para indicar al programa dónde se encuentra la configuración: 
```bash
export SOFTHSM2_CONF=/etc/softhsm2.conf
```

Por defecto, los tokens se almacenan en /var/lib/softhsm2/tokens. Esta carpeta puede ser un tanto restrictiva a la hora de crear y consultar los tokens. Crea una nueva carpeta "tokens" donde almacenaremos los tokens del softhsm. Yo la crearé en mi directorio home:
```bash
cd /home/daniel
mkdir tokens
```

Editamos el fichero de configuración de softhsm2 para indicarle la nueva localización de los tokens:
```bash
# SoftHSM v2 configuration file

directories.tokendir = /home/daniel/tokens
objectstore.backend = file

# ERROR, WARNING, INFO, DEBUG
log.level = ERROR

# If CKF_REMOVABLE_DEVICE flag should be set
slots.removable = false
```

Por último, generamos nuestros 3 tokens (a todos les damos contraseña 12345):
```bash
softhsm2-util --init-token --slot 0 --label fabric
softhsm2-util --init-token --slot 1 --label fabric1
softhsm2-util --init-token --slot 2 --label fabric2
```

### Reconstrucción de imagen de fabric-ca-server
Los binarios por defecto no dan soporte a pkcs#11, así que debemos de reconstruirlos:

Borramos las imágenes de la fabric CA que tenemos:
```bash
docker image ls
docker image rm <ID_IMAGE>
```

A continuación clonamos el repo de la fabric-ca y reconstruimos la imagen:
```bash
git clone https://github.com/hyperledger/fabric-ca.git # Clonamos repo de fabric-ca
cd fabric-ca # Entramos a la carpeta clonada
make docker GO_TAGS=pkcs11 # Ubuntu 20.04 Reconstruimos la imagen
make docker GO_TAGS=pkcs11 UBUNTU_VER=22.04 # Comando para Ubuntu 22.04
```

Tras estos pasos obtendremos verificamos la imagen recién creada con "docker image ls".

> **NOTA: PASOS ADICIONALES**
> Es probable que en Ubuntu 20.04 debas modificar el Dockerfile de la fabric-ca situado dentro de "fabric-ca/images/fabric-ca". Añade "RUN apt install -y libssl1.1" en la línea 53 justo debajo de "RUN apt update".

### Reconfiguración de Fabric CA
Las 3 Fabric CAs de la red guardan sus ficheros de configuración en:
- _/organizations/fabric-ca/org1/fabric-ca-server-config.yaml_
- _/organizations/fabric-ca/org2/fabric-ca-server-config.yaml_
- _/organizations/fabric-ca/ordererOrg/fabric-ca-server-config.yaml_

Hemos de modificar la sección BCCSP de estos ficheros para que quede así (cada CA apuntará a un token diferente cambiando el valor de Label):
```yaml
#############################################################################
# BCCSP (BlockChain Crypto Service Provider) section is used to select which
# crypto library implementation to use
#############################################################################
bccsp:
  default: PKCS11
  PKCS11:
    Library: /etc/hyperledger/softhsm/libsofthsm2.so
    Pin: 12345
    Label: fabric2
    hash: SHA2
    security: 256
    Immutable: false
```

### Montado de softhsm2 en docker de Fabric CA
- En la sección "volumes" del fichero montamos la librería libsofthsm2:
```yaml
volumes:
    [...]
    - /usr/lib/softhsm/libsofthsm2.so:/etc/hyperledger/softhsm/libsofthsm2.so
```
- También montamos la carpeta con los tokens para que el contenedor tenga acceso a ellos:
```yaml
volumes:
    [...]
    - /home/daniel/tokens:/etc/hyperledger/softhsm/tokens
```
- Hemos creado un fichero _hf-certification-practica/merca-chain/compose/softhsm2.conf_ con la siguiente configuración para indicar a la librería del interior del contenedor dónde tiene que buscar sos tokens:
```bash
# SoftHSM v2 configuration file

directories.tokendir = /etc/hyperledger/softhsm/tokens
objectstore.backend = file

# ERROR, WARNING, INFO, DEBUG
log.level = ERROR

# If CKF_REMOVABLE_DEVICE flag should be set
slots.removable = false
```
- Montamos este fichero de configuración al contenedor también:
```yaml
volumes:
    [...]
    - ./softhsm2.conf:/etc/hyperledger/softhsm/softhsm2.conf
```
- Por último, con la variable de entorno SOFTHSM2_CONF indicamos al contenedor dónde tiene el fichero de configuración del HSM.
```yaml
environment:
      [...]
      - SOFTHSM2_CONF=/etc/hyperledger/softhsm/softhsm2.conf
```
Todos estos cambios se pueden ver en detalle en los ficheros _compose/compose-ca.yaml_ y _compose/softhsm2.conf_.

> Algunas de las rutas descritas pueden variar en otros PCs: **/home/daniel...**

Reiniciamos la red y verificamos la aplicación de los cambios realizados con "docker ps -a" y "docker logs <CONTANIER_ID>" para cada CA.

![Image 4](./images/image4.PNG?raw=true "Fabric CA arrancando con PKCS11 (HSM)")

## Ejercicio 5
**Enunciado:** _Así mismo, explícales cómo configurar y utilizar el fabric-ca-client con el softHSM configurado. Registra, a modo de prueba, un usuario de nombre “merca-admin” y contraseña “merka-12345”, que tenga capacidad para registrar usuarios clientes y nuevos peers._

**Solución:**

Para estos efectos hemos creado una carpeta _fabric-ca-client_ dentro de _organizations_. Dentro de esta podemos encontrar un fichero _fabric-ca-client-config.yaml_ configurado para apuntar a la CA de la organización 1 y para usar el HSM. Véase la sección BCCSP de dicho fichero:
```yaml

bccsp:
  default: PKCS11
  PKCS11:
    Library: /usr/local/libsofthsm2.so
    Pin: 12345
    Label: fabric
    hash: SHA2
    security: 256
    Immutable: false
```

Al igual que ocurre con las imágenes, hemos de reconstruir el binario del fabric-ca-client para darle soporte para que use PKCS11. Seguimos estos pasos:
```bash
git clone https://github.com/hyperledger/fabric-ca.git # Clonamos repo de fabric-ca
cd fabric-ca # Entramos a la carpeta clonada
make fabric-ca-client GO_TAGS=pkcs11 # Reconstruimos el binario
```

El binario será generado en la carpeta _fabric-ca/bin_. Lo copiamos a nuestra carpeta _fabric-ca-client_, a la altura del fichero de configuración _fabric-ca-client-config.yaml_.

¡Ya podemos ejecutar los comandos para registro y enroll! Registro:
```bash
# Para interactuar como la boostrap identity de la organización 1 hacemos lo siguiente:
cd organizations/peerOrganizations/org1.example.com
export FABRIC_CA_CLIENT_HOME=$PWD
# Procedemos a registrar
cd ../../fabric-ca-client
./fabric-ca-client register --id.name merca-admin --id.secret merca-12345 --id.attrs '"hf.Registrar.Roles=peer,client"' --id.affiliation org1  --tls.certfiles ../../fabric-ca/org1/tls-cert.pem
```

![Image 5](./images/image5.PNG?raw=true "Registro de identidad")

En cuanto al enroll
```bash
# Para
export FABRIC_CA_CLIENT_HOME=$PWD
./fabric-ca-client enroll -u https://merca-admin:merca-12345@localhost:7054 -M $FABRIC_CA_CLIENT_HOME/msp --tls.certfiles ../fabric-ca/org1/tls-cert.pem 
```

![Image 6](./images/image6.PNG?raw=true "Enroll de identidad")

## Ejercicio 6
**Enunciado:** _Los merca-ingenieros de Merca-link consideran que tener herramientas de monitorización a su disposición agilizaría estas tareas de administración en futuras ocasiones. Despliega sendas instancias de Prometheus y Grafana para satisfacer estas necesidades._

**Solución:**

El primer paso es comprobar que los peers y los orderers están configurados correctamente para generar métricas para Prometheus y que el **operations service** esté activado. Deben de tener las siguientes variables de entorno configuradas en el fichero _compose/compose-test-network.yaml_:
```yaml
    # En el caso del peer:
    environment:
    [...]
      - CORE_OPERATIONS_LISTENADDRESS=peer0.org2.example.com:9445
      - CORE_METRICS_PROVIDER=prometheus
    [...]

    # En el caso del orderer:
    environment:
    [...]
      - ORDERER_OPERATIONS_LISTENADDRESS=orderer.example.com:9443
      - ORDERER_METRICS_PROVIDER=prometheus
    [...]
```

Configuradas estas variables podemos simplemente acceder a la carpeta _prometheus-grafana_ donde viene todo configurado para levantar una instancia de Prometheus y otra de Grafana, entre otros servicios de monitorización. Levantamos los servicios con "docker compose up -d" dentro de dicha carpeta. Esto resultará en:
- Prometheus levantado en _localhost:9090_. Podemos acceder sin contraseña.
- Grafana levandato en _localhost:3000_. Podemos acceder con las credenciales "admin" y "admin". Podemos ver aquellos dashboards que tengamos en la carpeta _/prometheus-grafana/grafana/provisioning/dashboards_ o bien crear o personalizar los nuestros propios.

![Image 7](./images/image7.PNG?raw=true "Dashboard de Prometheus")

![Image 8](./images/image8.PNG?raw=true "Dashboard de Grafana")


## Ejercicio 7
**Enunciado:** _Investiga sobre los despliegues distribuidos (esto es, en distintos nodos) de Hyperledger Fabric, y resume en unas pocas líneas tus recomendaciones y los principales puntos a tener en cuenta para que Merka-link despliegue su red de forma distribuida. No olvides referencias los artículos/blogs/páginas web que has consultado._

**Solución:**

Se trata de una pregunta bastante abierta que admite diferentes enfoques. En cualquier caso, los mínimos para un despliegue en varios nodos serían:
- Primero de todo, seleccionar nuestros hosts: máquinas virtuales para hacer pruebas, máquinas físicas, en cloud, enfoque multi cloud, mezcla de máquinas on premise y en cloud.
- Diseñar claramente qué componentes se desplegarán dónde, esto es, la topología de la red.
- También establecer conexión entre los hosts de manera segura: redes virtuales privadas, VPN, SD-WAN o similar.
- Otro mínimo es un orquestador de contenedores: en clase y en los apuntes hacemos uso de Docker Swarm. Usar Kubernetes es otro enfoque válido aunque requiere de una mayor curva de aprendizaje.
- Desplegar el contendor CLI de HF con las Fabric Tools ayudaría al mantenimiento de una red distribuida.

Otras alternativas:
- Plataformas Blockchain as a service (Kaleido).

Extras:
- Un plus de complejidad consistiría en hacer un despliegue mediante scripts y con herramientas de automatización como Ansible.
- Los servicios de monitorización serían clave en un despliegue de este tipo.

Algunos recursos interesantes al respecto:
- https://lists.hyperledger.org/g/fabric/topic/hyperledger_on_multiple_hosts/17549051
- https://github.com/saurabhsingh121/hyperledger-fabric-docker-swarm
- https://www.youtube.com/playlist?list=PLSBNVhWU6KjXDTwJlk8vI80kJy-rDG7JY
- https://kctheservant.medium.com/multi-host-deployment-for-first-network-hyperledger-fabric-v2-273b794ff3d
