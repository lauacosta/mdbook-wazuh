# Introducción

**Wazuh** es una plataforma de seguridad gratuita y de código abierto que unifica capacidades de protección [**XDR** (Extended detection and response)](https://en.wikipedia.org/wiki/Extended_detection_and_response) y [**SIEM** (Security information and event management)](https://en.wikipedia.org/wiki/Security_information_and_event_management). Protege cargas de trabajo en entornos virtualizados, contenedorizados y en la nube.

Ayuda a organizaciones e individuos a proteger sus datos contra amenazas de seguridad, ofreciendo:

- Análisis de logs  
- Detección de intrusiones y malware  
- Monitoreo de integridad del sistema de archivos  
- Evaluación de configuración  
- Detección de vulnerabilidades  
- Soporte para el cumplimiento normativo  

La plataforma está compuesta por un agente universal y tres componentes principales:

- [Wazuh Server](#servidor)
- [Wazuh Indexer](#indexador)
- [Wazuh Dashboard](#dashboard)

# Arquitectura

La arquitectura de Wazuh está basada en agentes que están siendo ejecutados en los endpoints a monitorear y envían información sobre seguridad al servidor central. Dispositivos sin agentes, como routers, switches, etc son soportados y pueden enviar información de logs vía Syslog, SSH o usando su propia API.

El servidor central decodifica y analiza la información que recibe y envía los resultados al Indexador para que sea indexada y almacenada.

Un ejemplo de cómo vendría a ser un deployment de Wazuh es el siguiente:
![Arquitectura de despliegue](/deployment-architecture1.png)

## Agente Wazuh - Comunicación con el servidor

El Agente envía constantemente eventos al servidor para su análisis y detección de amenazas. Para empezar a enviar esta informacion, el agente establece una conexión con el servicio del servidor para conexión de agente, el cual por defecto escucha al puerto 1514. El servidor decodifica y controla los eventos recibidos contra una serie de reglas, utilizando el motor de análisis. Aquellos eventos que infrinjan alguna regla son aumentados con información relacionada con la regla, como su ID y nombre. Los eventos pueden ser almacenados para su posterior procesamiento en uno o ambos de los siguientes archivos, de acuerdo a las reglas que infringen:
1. `/var/ossec/logs/archives/archives.json` contiene a todos los eventos sin importar si infringieron alguna regla o no.
2. `/var/ossec/logs/alerts/alerts.json` contiene solamente a los eventos que infringieron alguna regla con la suficiente prioridad de acuerdo a la configuración.

El protocolo de mensajes de Wazuh usa encriptación [AES](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard) ([Blowfish](https://en.wikipedia.org/wiki/Blowfish_(cipher)) es otra opción) por defecto, con bloques de 128 bits y llaves de 256 bits.

## Servidor Wazuh - Comunicación con el Indexador

El servido utiliza [Filebeat](https://www.elastic.co/beats/filebeat) para transmitir de manera segura información sobre las alertas y eventos al Indexador vía encriptación TLS. Filebeat monitorea la salida del servidor y la envía al indexador, el cual escucha por defecto en el puerto 9200/TCP. Una vez indexado, el usuario puede analizar y visualizar la información a través del Dashboard.                            

El módulo de detección de vulnerabilidad actualiza el inventario de vulnerabilidades. Además, genera alertas y provee información sobre las vulnerabilidades del sistema.

El Dashboard consulta a la Wazuh REST API (la cual escucha por defecto en el puerto 55000/TCP en el servidor) para mostrar información relacionada con la configuración y al status del servidor y de los agentes instalados. También puede modificar la configuración de los agentes y del servidor a través de llamadas a dicha API. Esta comunicación está encriptada con TLS y se debe estar autenticado con nombre de usuario y contraseña.                                                                                                                                                      

## Puertos requeridos

Varios servicios son utilizados para habilitar la comunicación entre los componentes de Wazuh. Esta es una lista de los puertos por defecto de estos servicios, los cuales pueden ser modificados si es necesario.

| Componente        | Puerto      | Protocolo         | Propósito                                             |
|------------------|-------------|-------------------|-------------------------------------------------------|
| Servidor Wazuh    | 1514        | TCP (por defecto) | Servicio de conexión de agentes                       |
| Servidor Wazuh    | 1514        | UDP (opcional)    | Servicio de conexión de agentes (desactivado por defecto) |
| Servidor Wazuh    | 1515        | TCP               | Servicio de registro de agentes                       |
| Servidor Wazuh    | 1516        | TCP               | Daemon del clúster de Wazuh                          |
| Servidor Wazuh    | 514         | UDP (por defecto) | Recolector Syslog de Wazuh (desactivado por defecto)  |
| Servidor Wazuh    | 514         | TCP (opcional)    | Recolector Syslog de Wazuh (desactivado por defecto)  |
| Servidor Wazuh    | 55000       | TCP               | API RESTful del servidor Wazuh                        |
| Indexador Wazuh   | 9200        | TCP               | API RESTful del indexador Wazuh                       |
| Indexador Wazuh   | 9300-9400   | TCP               | Comunicación del clúster del indexador Wazuh          |
| Panel de Wazuh    | 443         | TCP               | Interfaz web de usuario de Wazuh                      |

## Almacenamiento de datos archivados
Todos los eventos, tanto de alerta como no, son almacenados en archivos en el servidor, además de ser enviados al indexador. Estos archivos pueden estar escritos en formato JSON (.json), o texto plano (.log). Estos archivos son comprimidos diariamente y firmados utilizando MD5, SHA1, y SHA256 checksums. La estructura de directorios y nombre de archivos es la siguiente:                                             

``` bash
root@wazuh-manager:/var/ossec/logs/archives/2022/Jan# ls -l

total 176
-rw-r----- 1 wazuh wazuh 234350 Jan  2 00:00 ossec-archive-01.json.gz
-rw-r----- 1 wazuh wazuh    350 Jan  2 00:00 ossec-archive-01.json.sum
-rw-r----- 1 wazuh wazuh 176221 Jan  2 00:00 ossec-archive-01.log.gz
-rw-r----- 1 wazuh wazuh    346 Jan  2 00:00 ossec-archive-01.log.sum
-rw-r----- 1 wazuh wazuh 224320 Jan  2 00:00 ossec-archive-02.json.gz
-rw-r----- 1 wazuh wazuh    350 Jan  2 00:00 ossec-archive-02.json.sum
-rw-r----- 1 wazuh wazuh 151642 Jan  2 00:00 ossec-archive-02.log.gz
-rw-r----- 1 wazuh wazuh    346 Jan  2 00:00 ossec-archive-02.log.sum
-rw-r----- 1 wazuh wazuh 315251 Jan  2 00:00 ossec-archive-03.json.gz
-rw-r----- 1 wazuh wazuh    350 Jan  2 00:00 ossec-archive-03.json.sum
-rw-r----- 1 wazuh wazuh 156296 Jan  2 00:00 ossec-archive-03.log.gz
-rw-r----- 1 wazuh wazuh    346 Jan  2 00:00 ossec-archive-03.log.sum
```

# Componentes

 ![componentes de wazuh y flujo de datos](wazuh-components-and-data-flow1.png)

## Indexador
Motor de búsqueda y análisis de texto completo. Este componente indexa y almacena la información generada por el servidor.

## Servidor
El servidor analiza los datos recibidos de los agentes. Los procesa, decodificándolos y controlándolo con reglas, utilizando información sobre amenazas para buscar indicadores de compromiso (IOCs) bien conocidos.

Este componente central también se utiliza para gestionar los agentes, configurándolos y actualizándolos remotamente cuando es necesario.

## Dashboard
Interfaz web para visualización y análisis de datos. Incluye paneles predefinidos para búsqueda de amenazas, cumplimiento normativo (por ejemplo, PCI DSS, GDPR, CIS, HIPAA, NIST 800-53), aplicaciones vulnerables detectadas, datos de monitoreo de integridad de archivos, resultados de evaluación de configuración, eventos de monitoreo de infraestructura en la nube, y otros. También se utiliza para gestionar la configuración de Wazuh y monitorear su estado.

## Agentes
Los agentes Wazuh se instalan en endpoints como laptops, escritorios, servidores, instancias en la nube o máquinas virtuales. Proporcionan capacidades de prevención, detección y respuesta ante amenazas. 

Además de las capacidades de monitoreo basadas en agentes, la plataforma Wazuh puede monitorear dispositivos sin agentes como firewalls, switches, routers o IDS de red, entre otros.

# Casos de uso



