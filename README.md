
#### Ejercicio 1: Despliegue de la red
1. **Problema**: Algunos contenedores no se levantan correctamente.
2. **Solución**:
   - Hacer ejecutables los scripts necesarios usando `sudo chmod +x`.
   - Instalar los binarios de Fabric con `install-fabric.sh b`.
   - Desplegar la red con `./network.sh up -ca`.
   - Identificar y solucionar problemas de contenedores caídos (peer y orderer) debido a errores de rutas de certificados y typos en configuraciones.
   - Corregir las rutas y reiniciar la red.

#### Ejercicio 2: Despliegue del chaincode
1. **Problema**: El chaincode “merca-chaincode” no se despliega correctamente.
2. **Solución**:
   - Crear el canal y desplegar el chaincode.
   - Identificar errores en los logs de los contenedores del chaincode.
   - Corregir errores de sintaxis en el código del chaincode.
   - Actualizar o reinstalar el chaincode.

#### Ejercicio 3: Configuración de logs
1. **Requerimientos**:
   - Logs de orderers en formato JSON.
   - Logs de peers con fecha y hora al final y nivel de logging en DEBUG por defecto.
   - Logging específico para gossip en WARNING y chaincode en INFO.
2. **Solución**:
   - Configurar las variables de entorno adecuadas en el fichero docker-compose.
   - Reiniciar la red y verificar los cambios en los logs.

#### Ejercicio 4: Securización con HSM
1. **Problema**: Claves criptográficas almacenadas en texto plano.
2. **Solución**:
   - Instalar y configurar SoftHSM2.
   - Generar tokens para las CAs.
   - Reconstruir la imagen de fabric-ca-server con soporte PKCS#11.
   - Configurar las CAs para usar SoftHSM.
   - Montar la librería y tokens de SoftHSM en los contenedores de Fabric CA.
   - Reiniciar la red y verificar los cambios.

#### Ejercicio 5: Uso de fabric-ca-client con HSM
1. **Problema**: Necesidad de configurar y usar fabric-ca-client con SoftHSM.
2. **Solución**:
   - Configurar fabric-ca-client para usar HSM.
   - Reconstruir el binario fabric-ca-client con soporte PKCS#11.
   - Registrar y enrolar un usuario de prueba “merca-admin”.

#### Ejercicio 6: Monitorización con Prometheus y Grafana
1. **Problema**: Necesidad de herramientas de monitorización.
2. **Solución**:
   - Configurar peers y orderers para generar métricas para Prometheus.
   - Levantar instancias de Prometheus y Grafana.
   - Verificar la configuración y el acceso a los dashboards de monitorización.

#### Ejercicio 7: Despliegue distribuido de Hyperledger Fabric
1. **Requerimientos**: Desplegar la red en distintos nodos.
2. **Recomendaciones**:
   - Selección de hosts (máquinas virtuales, físicas, cloud).
   - Diseño de la topología de la red.
   - Establecimiento de conexiones seguras entre hosts.
   - Uso de un orquestador de contenedores (Docker Swarm o Kubernetes).
   - Consideración de servicios de monitorización.
   - Exploración de opciones como Blockchain as a Service y herramientas de automatización como Ansible.
3. **Recursos recomendados**: Links a artículos y repositorios relevantes para despliegues distribuidos.

