# SNMP_Exporter_Tutorial
A fast guide to understand the basic concepts under SNMP Exporter Prometheus plugin. This guide is a simple reference that consists of basic SNMP concepts, Prometheus concepts and SNMP+Prometheus concepts with code examples

# Index
1. [SNMP](#snmp)

    1.1. [Referencias de interés](#referencias-de-interes)

2. [Prometheus](#prometheus)

    2.1. [Endpoints en Prometheus](#endpoints-en-prometheus)

    2.2. [Configuración de prometheus.yml](#configuración-de-prometheusyml)

3. [Exporter SNMP Prometheus](#exporter-snmp-programa-cliente-gestor-snmp)

    3.1. [Configuración de snmp.yml](#cómo-configurar-snmpyml-en-el-exporter-snmp)

    3.2. [Interactuar con el Exporter](#cómo-interactuar-con-el-exporter-en-ejecución)
---
---
---

# SNMP
Se trata de un protocolo para la gestión de equipos en la red que
permite extraer sus métricas. Funciona mediante:

-   **Agente o Servidor SNMP**: equipo del que se extraen las métricas.
    Ejemplos: servicio snmpd de Linux. Se puede ejecutar en localhost,
    en el puerto 161 por defecto.

-   **Gestor o Cliente SNMP**: equipo que solicita al agente las
    métricas. Ejemplos: snmp_exporter o comandos de Linux (snmpwalk,
    snmpget etc).

SNMP cuenta con tres versiones principalmente:
-   V1
-   V2/V2C (introduce el community name)
-   V3 (con autenticación user/password, por ejemplo)

Los equipos gestionados deben \"querer ser gestionados\"; es decir, no
se pueden obtener métricas de cualquier dispositivo de red, sino solo de
aquellos que ejecuten un agente SNMP.

Aunque un equipo ejecute un agente SNMP, no se puede obtener cualquier
métrica, sino solo aquellas que exponga el agente públicamente para el
gestor. Por un lado, el agente tendrá una lista de métricas que desea
exponer y, por el otro, el gestor tendrá otra lista de métricas que le pedirá al
agente. Ambas listas no tienen por qué ser iguales. De hecho, el gestor
pedirá varias métricas al agente y puede ser que algunas de ellas no las
pueda obtener, porque el agente no las expone.

```
Por ejemplo, en el caso de snmpd como agente SNMP, la lista de métricas que se desea
exponer se puede configurar en /etc/snmp/snmpd.conf
```

Las métricas se denominan también objetos (objeto sysUpTime, ifName,
bandwidth etc) y cada una tiene un **OID único**. Existe un árbol
universal de métricas donde se puede explorar todo lo que se puede
gestionar con SNMP.

*Ejemplo: .1.3.6.1.2.1.5 (valor inventado) = métrica bandwidth*

Podemos obtener la lista de métricas que expone un agente concreto
realizando un ```snmpwalk```:

```
snmpwalk -v <version_snmp (e.g. 2c)> -c <community_name> <IP_agente> <OID o metric_name>
```

El comando ```snmpwalk``` obtendrá todos los OIDs dentro del OID especificado que el
agente está exponiendo.

Dentro del árbol universal de métricas (OIDs) se encuentran las
**MIBs**. Las MIBs permiten la traducción de OIDs a nombres de métricas
legibles por el ser humano. Una MIB es un subárbol dentro del árbol
universal que contiene varias métricas dentro, cada una con su OID. Será
habitual que, al especificar en el gestor SNMP una OID que se desea
obtener del agente, sea también necesario especificar su MIB. Incluso
será necesario descargarla ya sea en formato ```.mib``` como ```.txt```, o bien
copiar y pegar su contenido a un fichero ```.mib``` o ```.txt``` que creemos en
local.

## Referencias de interés

-   **Oidref**: Permite obtener la información de una OID
    concreta. Nos dará la MIB a la que pertenece esa métrica. Por
    ejemplo, si queremos obtener info. de la métrica .1.3.6.1.9.8.7.6
    (valor inventado), accedemos al endpoint:
    
    [oidref.com/.1.3.6.1.9.8.7.6](https://oidref.com/.1.3.6.1.9.8.7.6)

-   [**Circitor**](https://www.circitor.fr/Mibs/Mibs.php): Permite descargar las MIBs.

-   [**SNMPLink**](http://www.snmplink.org/): Permite explorar el árbol universal de OIDs
    y el árbol de las MIBs. También permite descargar las MIBs.



---
---
---

# Prometheus

En resumen, Prometheus es una Base de Datos orientada a Series Temporales y métricas. Las series
temporales se almacenan en la base de datos con un formato tipo:

```
metricName{label1=X, label2=Y...}
```

*Ejemplos:*
```
Temperature{server: "localhost:0001", process: "my_service1"}
Temperature{server: "localhost:0001", process: "my_service2"}
```

Prometheus obtiene series temporales de métricas especificando un
**endpoint** concreto (llamado instancia). Ese endpoint típicamente será
HTTP y debe devolver los datos a Prometheus en un formato que Prometheus
comprenda.

Los endpoints que queremos rascar (esto es, de los que queremos obtener métricas)
se especifican en el fichero ```prometheus.yml```
(típicamente localizado en ```/etc/prometheus/prometheus.yml```), en la sección:

```
scrape_configs:
    static_configs:
        targets: <---
```

Ejemplo de ```prometheus.yml```:

```
scrape_configs:
  - job_name: 'prometheus-itself'
    scrape_interval: 5s
    static_configs:
      - target_configs: ['localhost:9090']
```
_(En este caso, estamos creando un job que escrapea cada 5 segundos métricas del endpoint localhost:9090)_

## Endpoints en Prometheus

Los endpoints que podemos rascar en Prometheus deben devolver datos a
Prometheus en un formato concreto. Ese endpoint puede ser un Script en
ejecución en un puerto concreto de nuestra máquina local, por ejemplo.
Ese Script se puede obtener de diversos repositorios de GitHub y
básicamente se denomina **exporter**. Hay muchos [exporters](https://prometheus.io/docs/instrumenting/exporters/) a nuestra disposición,
entre los que está el [exporter de SNMP](https://github.com/prometheus/snmp_exporter).

La idea será ejecutar el exporter (e.g.: ```./snmp_exporter```, donde
```snmp_exporter``` es el fichero ```.sh``` o Script que ejecuta el endpoint) y
después configurar ```prometheus.yml``` para que Prometheus rasque ese
endpoint.


## Configuración de prometheus.yml

Los endpoints, por ser HTTP, pueden requerir _Query strings_, por ejemplo,
o _Request paths_ dentro del directorio raíz.

Ejemplo:
```
localhost:9090/example?x=1&y=2
```

Un problema común es cómo especificar este tipo de endpoints en
```prometheus.yml```, pues, en la sección ```targets:```, Prometheus no admite
especificar nada que no sea en un formato ```IP:port```.


### **Targets estáticos con request path**

Si se desea especificar el endpoint ```localhost:9090/example```, sin query
strings, simplemente esto requiere especificar el ```metrics_path:```, sin
configuraciones adicionales:

```
scrape_configs:
  - job_name: 'prometheus-itself'
    metrics_path: '/example' <---
    scrape_interval: 5s
    static_configs:
      - target_configs: ['localhost:9090']
```

### **Targets dinámicos**

#### Request path dinámico:

Suponiendo que genéricamente se desea rascar los siguientes
endpoints dentro del mismo job:

- ```localhost:9090/snmpv1```

- ```localhost:9090/snmpv2```

- ```localhost:9090/snmpv3```

En este caso, cada endpoint posee un _Request path_ diferente, por lo
que asignar un valor único no es posible. Por tanto, se utiliza el
**relabeling** de Prometheus. El **relabeling** de Prometheus sirve
para:

-   **Especificar endpoints que no sean del tipo ```IP:port``` en la sección
    ```targets:```**, pero siempre y cuando después de especificarlo hagamos
    un arreglo con el relabeling para que, cuando Prometheus rasque
    esos endpoints de ```targets:```, automáticamente y antes del rascado
    descomponga la cadena de caracteres y guarde en ```__address__```
    la parte ```IP:port``` y en ```__metrics_path__``` la parte
    ```/loquesea```.
    
    Las variables (o etiquetas) ```__address__``` y
    ```__metrics_path__``` en ```prometheus.yml``` son variables reconocidas de Prometheus y basta
    con que tengan el formato adecuado en el momento del escrapeo para que funcione el job (esto es,
    ```__address__``` debe ser tipo ```IP:port``` y ```__metrics_path__``` debe
    contener el _Request path_). De hecho, esas son las variables que
    realmente utiliza Prometheus para rascar.

-   **Especificar que, cuando Prometheus rasque el endpoint
    ```localhost:9090``` especificado correctamente en ```targets:```,
    automáticamente se concatene una string (```/loquesea```)**.

Por tanto, se debe utilizar el primer caso de uso del relabeling:

```
scrape_configs:
  - job_name: 'prometheus-itself'
    scrape_interval: 5s
    static_configs:
      - targets:
        - 'localhost:9090/snmpv1'
        - 'localhost:9090/snmpv2'
        - 'localhost:9090/snmpv3'
    
    relabel_configs: # Justo antes del momento de rascado (e.g. hacia localhost:9090/snmpv2), sucede lo siguiente:
      - source_labels: [__address__]                                                                  # En este momento, __address__ = 'localhost:9090/snmpv2'
        regex: 'Expresión regular aplicada a source_labels para coger solo la parte __metrics_path__'
        target_label: __metrics_path__                                                                # __metrics_path__ <=== /snmpv2

      - source_labels: [__address__]                                                                  # En este momento, __address__ = 'localhost:9090/snmpv2' todavía
        regex: 'Expresión regular aplicada a source_labels para coger solo la parte IP:port'
        target_label: __address__                                                                     # __address__      <=== localhost:9090
```

Si queremos añadir una _Query string_, podemos hacerlo, pues el
**relabeling** se encarga de guardar en ```__address__``` el valor
```IP:port``` y en ```__metrics_path__``` el valor del _Request path_. Para añadir
_Query string_ parameters (e.g.: ```localhost:9090/example?x=3```):

```
scrape_configs:
  - job_name: 'prometheus-itself'
    scrape_interval: 5s
    static_configs:
      - targets:
        - 'localhost:9090/snmpv1'
        - 'localhost:9090/snmpv2'
        - 'localhost:9090/snmpv3'
    
    params:
        - x: ['3']

    relabel_configs: # Justo antes del momento de rascado (e.g. hacia localhost:9090/snmpv2), sucede lo siguiente:
      - source_labels: [__address__]                                                                  # En este momento, __address__ = 'localhost:9090/snmpv2'
        regex: 'Expresión regular aplicada a source_labels para coger solo la parte __metrics_path__'
        target_label: __metrics_path__                                                                # __metrics_path__ <=== /snmpv2

      - source_labels: [__address__]                                                                  # En este momento, __address__ = 'localhost:9090/snmpv2' todavía
        regex: 'Expresión regular aplicada a source_labels para coger solo la parte IP:port'
        target_label: __address__                                                                     # __address__      <=== localhost:9090

        # En este momento, __address__ = localhost:9090. Tenerlo en cuenta si fuéramos a usarla más abajo
```

#### Query string dinámica:

Si la _Query string_ es diferente según el endpoint (por ejemplo, el
_Request path_ ```/snmpv1``` requiere parámetros distintos a los demás), podemos
usar el **relabeling** para que, cuando ```__metrics_path__``` cumpla una expresión regular (por ejemplo,
que sea = \"/snmpv1\", se cree una nueva label que siempre debe tener el nombre
reconocido ```__param_<key>``` (donde ```<key> = nombre del parámetro de la query string```) que añada un parámetro a la URL llamado ```<key>``` con valor \"el que
sea\". Por ejemplo:

```
scrape_configs:
  - job_name: 'prometheus-itself'
    scrape_interval: 5s
    static_configs:
      - targets:
        - 'localhost:9090/snmpv1'
        - 'localhost:9090/snmpv2'
        - 'localhost:9090/snmpv3'
    
    params:
        - x: ['3']

    relabel_configs: # Justo antes del momento de rascado (e.g. hacia localhost:9090/snmpv2), sucede lo siguiente:
      - source_labels: [__address__]                                                                  # En este momento, __address__ = 'localhost:9090/snmpv2'
        regex: 'Expresión regular aplicada a source_labels para coger solo la parte __metrics_path__'
        target_label: __metrics_path__                                                                # __metrics_path__ <=== /snmpv2

      - source_labels: [__address__]                                                                  # En este momento, __address__ = 'localhost:9090/snmpv2' todavía
        regex: 'Expresión regular aplicada a source_labels para coger solo la parte IP:port'
        target_label: __address__                                                                     # __address__      <=== localhost:9090
        
        # En este momento, __address__ = localhost:9090. Tenerlo en cuenta si fuéramos a usarla más abajo
      
      - source_labels: [__metrics_path__]
        regex: '/snmpv1'
        target_label: __param_x
        replacement: ['3'] # O el valor que queramos mandar
      
      - source_labels: [__metrics_path__]
        regex: 'Expresión regular aplicada a source_labels para comprobar que sea = /snmpv2'
        target_label: __param_x
        replacement: ['3'] # O el valor que queramos mandar
      - source_labels: [__metrics_path__]
        regex: 'Expresión regular aplicada a source_labels para comprobar que sea = /snmpv2'
        target_label: __param_y
        replacement: ['4'] # O el valor que queramos mandar
      
      - source_labels: [__metrics_path__]
        regex: 'Expresión regular aplicada a source_labels para comprobar que sea = /snmpv3'
        target_label: __param_x
        replacement: ['3'] # O el valor que queramos mandar
```

De esta manera y para el ejemplo, si el _Request path_ es ```/snmpv1```, automáticamente se
añadirá ```?x=3``` a la URL. Si es ```/snmpv2```, se añadirá ```?x=3&y=4```. Si es ```/snmpv3```, se añade también ```?x=3```.

Por último, algunos endpoints requieren un tiempo largo para ser
rascados, por lo que conviene:

1.  Configurar un ```scrape_interval``` alto, que define el período de muestreo.

2.  Configurar un ```scrape_timeout``` alto, para que Prometheus espere un
    tiempo largo a recibir los datos. Este valor debe ser siempre menor
    que ```scrape_interval```, como es lógico.

```
scrape_configs:
  - job_name: 'prometheus-itself'
    scrape_interval: 5s
    static_configs:
      - targets:
        - 'localhost:9090/snmpv1'
        - 'localhost:9090/snmpv2'
        - 'localhost:9090/snmpv3'
    
    params:
        - x: ['3']

    scrape_interval: 1m
    scrape_timeout: 59s

    relabel_configs: # Justo antes del momento de rascado (e.g. hacia localhost:9090/snmpv2), sucede lo siguiente:
      - source_labels: [__address__]                                                                  # En este momento, __address__ = 'localhost:9090/snmpv2'
        regex: 'Expresión regular aplicada a source_labels para coger solo la parte __metrics_path__'
        target_label: __metrics_path__                                                                # __metrics_path__ <=== /snmpv2

      - source_labels: [__address__]                                                                  # En este momento, __address__ = 'localhost:9090/snmpv2' todavía
        regex: 'Expresión regular aplicada a source_labels para coger solo la parte IP:port'
        target_label: __address__                                                                     # __address__      <=== localhost:9090
        
        # En este momento, __address__ = localhost:9090. Tenerlo en cuenta si fuéramos a usarla más abajo
      
      - source_labels: [__metrics_path__]
        regex: '/snmpv1'
        target_label: __param_x
        replacement: ['3'] # O el valor que queramos mandar
      
      - source_labels: [__metrics_path__]
        regex: 'Expresión regular aplicada a source_labels para comprobar que sea = /snmpv2'
        target_label: __param_x
        replacement: ['3'] # O el valor que queramos mandar
      - source_labels: [__metrics_path__]
        regex: 'Expresión regular aplicada a source_labels para comprobar que sea = /snmpv2'
        target_label: __param_y
        replacement: ['4'] # O el valor que queramos mandar
      
      - source_labels: [__metrics_path__]
        regex: 'Expresión regular aplicada a source_labels para comprobar que sea = /snmpv3'
        target_label: __param_x
        replacement: ['3'] # O el valor que queramos mandar
```

---
---
---

# Exporter SNMP (Programa cliente Gestor SNMP)

El Exporter SNMP es un Script que debemos obtener del [repositorio de
GitHub](https://github.com/prometheus/snmp_exporter). Como se explicó [anteriormente](#endpoints-en-prometheus), Prometheus deberá rascar el proceso del
exporter mientras estemos ejecutando ese Script.


El Script del Exporter realmente se encuentra en la [siguiente URL](https://github.com/prometheus/snmp_exporter/releases)).
Debemos descargar los ficheros binarios según nuestra arquitectura de
procesador y Sistema Operativo y descomprimir el archivo. Al
descomprimirlo, obtendremos:

-   El Script del exporter. Se recuerda que el Exporter
    SNMP es el cliente Gestor SNMP realmente.

-   El fichero ```snmp.yml```, que es donde se configura la
    lista de OIDs que pedir al agente SNMP. Este fichero requiere
    configuraciones adicionales que se explican a continuación, por lo
    que no conviene modificarlo manualmente.

El directorio en el que se descargue y ejecute el Script del Exporter no tiene por
qué ser el mismo que el directorio donde clonemos el repositorio de
GitHub de SNMP Prometheus Exporter. El primero es el gestor propiamente dicho. El segundo, en cambio, es el repositorio que nos permite configurar el gestor.


## Cómo configurar snmp.yml en el Exporter SNMP

En este fichero se especificará la lista de OIDs que queremos pedir al
servidor o agente SNMP. Sin embargo, no se debería modificar a mano,
sino que, en su lugar, existe un [código generador](https://github.com/prometheus/snmp_exporter/tree/main/generator)
que lo hace automáticamente a partir de la lista de OIDs que queramos y
de las MIBs.

### Generator de snmp.yml

El código del generador se encarga de escribir por nosotros el fichero
```snmp.yml``` a partir de dos inputs principales:

-   Las MIBs que haya dentro del directorio ```/generator/mibs```

-   La lista de OIDs que queremos obtener, que estarán especificadas
    dentro de ```/generator/generator.yml```

El código del generador se encarga de comprobar que cada OID que hayamos
especificado dentro del ```/generator/generator.yml``` case con las MIBs de ```/generator/mibs```, es decir, que
exista ese OID dentro de alguna MIB. Para
ello, parsea las MIBs y realiza la comprobación.

#### Descarga las MIBs

El primer paso sería descargar las MIBs deseadas. Es importante
considerar que las MIBs suelen tener dependencias unas con otras, por lo
que, si descargamos una MIB en concreto, lo más seguro es que sea
necesario descargar sus dependencias también (esto es, las MIBs de las
que depende). Podemos ver las dependencias de una MIB si vemos el
contenido de la MIB, típicamente en la sección IMPORT. Por ejemplo:

```
IMPORTS
    MODULE-IDENTITY, OBJECT-TYPE, Counter32, Integer32, TimeTicks,
        NOTIFICATION-TYPE, OBJECT-IDENTITY,
        mib-2 FROM SNMPv2-SMI                             -- [RFC2578]
    TEXTUAL-CONVENTION FROM SNMPv2-TC                     -- [RFC2579]
    MODULE-COMPLIANCE, OBJECT-GROUP, NOTIFICATION-GROUP
        FROM SNMPv2-CONF                                  -- [RFC2580]
    hrDeviceIndex, hrStorageIndex FROM HOST-RESOURCES-MIB -- [RFC2790]
    InterfaceIndexOrZero FROM IF-MIB                      -- [RFC2863]
    PrtCoverStatusTC, PrtGeneralResetTC, PrtChannelTypeTC,
        PrtInterpreterLangFamilyTC, PrtInputTypeTC, PrtOutputTypeTC,
        PrtMarkerMarkTechTC, PrtMarkerSuppliesTypeTC, PrtConsoleColorTC,
        PrtConsoleDisableTC, PrtMediaPathTypeTC, PrtAlertGroupTC,
        PrtAlertTrainingLevelTC, PrtAlertCodeTC
        FROM IANA-PRINTER-MIB
    IANACharset FROM IANA-CHARSET-MIB; <------------
```

Otra opción es descargar las MIBs que vienen por defecto en el
repositorio utilizando el comando siguiente dentro del directorio ```/generator```:

```
make generator mibs
```

Esto último es útil porque es muy posible que algunas de las MIBs que descarga
este comando sean dependencias de las MIBs que hayamos descargado de
Internet, así que conviene realizar ambas cosas: descargar las que
queramos de Internet y descargar las "default" con el comando ```make
generator mibs```.

Una vez descargadas las MIBs, conviene ejecutar un comprobador de las
MIBs utilizando el siguiente comando:

```
sudo docker run --rm -v "${PWD}:/opt/" prom/snmp-generator:v0.24.1 parse_errors
```

Si no hay logs de error, podemos continuar configurando el fichero
```generator.yml```.

#### Configura el Generator

##### AUTENTICACIÓN

Para configurar el fichero ```generator.yml``` (ver plantilla [aquí](https://github.com/prometheus/snmp_exporter/tree/main/generator#file-format)), en primer lugar se deberá
añadir el método de autenticación contra el agente SNMP. Por ejemplo,
para la versión 2 de SNMP, basta con establecer un ```community_name```,
aunque la autenticación dependerá del agente:

```
auths:
  mi_autenticacion:
    version: 2
    community: nombre_comunidad
```

En ```mi_autenticacion``` pondremos el nombre que queramos de la autenticación.
Este nombre será importante, ya que es el que se debe enviar en la query
string cuando el Exporter trate de obtener datos del agente (se verá [más adelante](#cómo-interactuar-con-el-exporter-en-ejecución)).

##### MIBS Y OIDS

A continuación, se deberá configurar la lista de OIDs que queramos pedir
al agente. Dado que especificar todas y cada una de las OIDs no es
viable (a menudo se quieren monitorizar miles de métricas), es más común
especificar la OID padre desde la que hacer un ```snmpwalk``` (e.g. si se
especifica ```1.3.6.1.4.1```, se recorrerán todas las OIDs
dentro de esa OID, es decir, ```1.3.6.1.4.1.X```):

```
auths:
  mi_autenticacion:
    version: 2
    community: nombre_comunidad

modules:
  mi_mib:
    walk:
      - 1.3.6.1.4.1

    max_repetitions: 25  # How many objects to request with GET/GETBULK, defaults to 25.
                         # May need to be reduced for buggy devices.
    retries: 3   # How many times to retry a failed request, defaults to 3.
    timeout: 5s  # Timeout for each individual SNMP request, defaults to 5s.

    lookups:
     - source_indexes: [index_object_name]
       lookup: object_name_column
```

El nombre del módulo (```mi_mib``` en el código de ejemplo) será el que
queramos, pero este nombre será importante, ya que es el que se debe
enviar en la query string cuando el Exporter trate de obtener datos del
agente (se verá [más adelante](#cómo-interactuar-con-el-exporter-en-ejecución)).

En una tabla SNMP, existe un objeto índice (o varios) para indexar los
elementos dentro de la tabla. Por defecto, cuando se obtiene un valor
concreto de la tabla, SNMP, en lugar de devolver el nombre de la métrica
pedida, devuelve el nombre del objeto índice, lo que puede ser confuso,
pues no sabemos a qué métrica se refiere. Para ello, en la sección
```lookups``` será posible especificar (I) el nombre del objeto índice de
la tabla (u objetos índice) y (II) el nombre de la columna o métrica en
cuestión (deben ser exactamente esos nombres los que debemos poner en
```generator.yml```). De esta manera, SNMP devolverá tanto el nombre del
objeto índice como el nombre de la métrica y Prometheus guardará la
serie temporal con ambas etiquetas.

Por ejemplo, si tenemos la siguiente tabla:

| index_object_name | cpu_usage |
|:-----------------:|:---------:|
| 0 | 0.34 |
| 1 | 0.28 |
| 2 | 0.56 |
| 3 | 0.11 |
| 4 | 0.78 |

Al realizar un SNMP walk sobre el OID de la tabla, si no se especifica ningún ```lookups```, el resultado sería:

```
cpu_usage{index_object_name="0"} 0.34
cpu_usage{index_object_name="1"} 0.28
cpu_usage{index_object_name="2"} 0.56
cpu_usage{index_object_name="3"} 0.11
cpu_usage{index_object_name="4"} 0.78
```

Aunque la métrica puede ser suficientemente descriptiva (```cpu_usage```), tal vez sea de interés añadir una label con su nombre. Si se especifican ```lookups```:

```
cpu_usage{index_object_name="0", cpu_usage="0.34"} 0.34
cpu_usage{index_object_name="1", cpu_usage="0.28"} 0.28
cpu_usage{index_object_name="2", cpu_usage="0.56"} 0.56
cpu_usage{index_object_name="3", cpu_usage="0.11"} 0.11
cpu_usage{index_object_name="4", cpu_usage="0.78"} 0.78
```

En la sección ```overrides:``` de ```generator.yml```
se pueden aplicar ciertas transformaciones al valor de la métrica (por
ejemplo, si la CPU viene entre 0 y 1 y queremos que esté entre 0 y 100 +
offset, o bien si un cierto dato devuelve UP/DOWN como strings y
queremos transformarlo a booleano True/False, o si un cierto dato viene
en float y queremos transformarlo a int...). Podemos ya sea sobrescribir
la métrica o bien crear una nueva métrica en base a la métrica original
con ```ignore:```.

También es posible aplicar filtrados en el SNMP walk. Esto es muy útil,
porque cuando especificamos el SNMP walk en la sección ```walk:```, quizás
no queramos obtener todos los objetos dentro del OID general
(es decir, ```1.3.6.1.4.1.X```). Por tanto, se pueden configurar filtros para
especificar qué OIDs queremos obtener dentro del OID general sobre el
que se hace el walk:

```
filters:
    static:
      - targets:
        - 1.3.6.1.4.1
        indices: ["3", "6", "9"] # Solo se desea obtener el OID 1.3.6.1.4.1.3, 1.3.6.1.4.1.6 y 1.3.6.1.4.1.9
```
 
Por otro lado, existen filtros dinámicos aplicados a tablas SNMP.
Suponiendo que tenemos una tabla tipo:


|ifNumber         | ifState       | Input_bits        | ifAddress      |
|:---------------:|:-------------:|:-----------------:|:--------------:|
|0                | UP            | 3456234           | A              |
|1                | UP            | 24356456          | B              |
|2                | DOWN          | 67                | C              |
|3                | UP            | 2355              | D              |


_Siendo el OID de la tabla = 1.3.6.1.999 por ejemplo._

Y queremos solamente visualizar los Input_bits de los enlaces con estado
UP, entonces podemos aplicar un filtro dinámico sobre la columna
Input_bits basado en la columna ifState tal que solo se muestren valores
en UP como:

```
dynamic:
    - oid: 1.3.6.1.999.3    # Columna Input_bits
    targets:
        - "1.3.6.1.999.2"   # Columna ifState
    values: ["UP"]          # Valor de filtrado
```

Por último, el fichero ```generator.yml``` posee otras configuraciones de
timeout hacia el agente, max_repetitions, retries...

#### Crear snmp.yml

Una vez configurado el fichero ```generator.yml```, será necesario volcar el
resultado a ```snmp.yml``` mediante:

```
make generate
```

_¡NOTA!: El fichero resultante se vuelca al directorio /generator/snmp.yml (no a /snmp.yml)._

Este fichero se deberá transferir al directorio donde se encuentre el
Script .sh del Exporter. Si el Script estuviera actualmente en
ejecución, se deberá detener y volver a lanzar para aplicar los cambios.

 
## Cómo interactuar con el Exporter en ejecución

Mientras el Script del Exporter está en ejecución (se lanzará en el
puerto ```9116``` por defecto, aunque el puerto se puede
configurar), tendremos listo el Gestor SNMP. Sin embargo, es necesario
especificar:

-   A quién (a qué agente) realizar las peticiones SNMP

-   Cuál es la community

-   (Opcional) Qué MIB (módulo) queremos obtener del agente - Permite
    obtener solo las métricas de una MIB concreta dentro de todas las
    métricas que le pide el gestor.

Para ello, se especifica en forma de _Query string_ hacia el endpoint
```/snmp```. En este paso es donde tendremos que utilizar los nombres que
pusimos en ```generator.yml```.

Suponiendo que el gestor se ejecuta en la IP ```IP_gestor``` y puerto ```puerto_gestor``` y el agente en ```IP_agente```:

```
IP_gestor:puerto_gestor/snmp?target=IP_agente&auth=mi_autenticacion&module=mi_mib
```

### Cómo configurar prometheus.yml para que interactúe con el Exporter

Sabiendo cómo interactuar con el exporter, el último paso es crear el
job de Prometheus que rasque el endpoint HTTP donde se está ejecutando. Para
ello, debemos configurar que el job acceda a ```/snmp``` y mande las variables
que sean necesarias en la query string, como se explicó con
anterioridad:

```
...

scrape_configs:
    - job_name: snmp-tests
      metrics_path: '/snmp'
      scrape_interval: 1m
      scrape_timeout: 59s
      static_configs:
      - targets:
        - 'IP_gestor:puerto_gestor/snmp'
      params:
      target: ['IP_agente']

      relabel_configs:
      - source_labels:
        - __address__
        regex: '[^/]+(/.*)'
        target_label: __metrics_path__

      - source_labels:
        - __address__
        regex: '([^/]+)/.*'
        target_label: __address__
        
...
```
