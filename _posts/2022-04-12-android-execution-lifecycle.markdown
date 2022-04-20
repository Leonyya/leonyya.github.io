---
layout: post
title:  "Ciclo de vida de ejecución en Android, notas de repaso"
date:   2022-04-12 12:58:29
categories: android
---

ART o Android Runtime es la máquina virtual *sucesor del Dalvik* que ejecuta el formato del ejecutable Dalvik (*Dex bytecode*), cada archivo *.java* fuente del APK es compilado a *.class* (*Java Bytecode*), posteriormente por cada uno de los archivos de clase se genera un archivo *.dex* específicamente el llamado "*classes.dex*", que es el formato ejecutable de Dalvik, para después colocarlo junto a *AndroidManifest.xml*, Assets, Resources y Libs. A continuación la estructura de un archivo apk después de descomprimir el zipfile: 

{% highlight txt %}
|-- AndroidManifest.xml
|-- META-INF // Signature Data
|   |-- CERT.RSA
|   |-- CERT.SF
|   `-- MANIFEST.MF
|-- classes.dex //  java byte code file generated after the compilation
|-- res // resource files
|   |-- drawable
|   |   `-- icon.png
|   `-- layout
|       `-- main.xml
`-- resources.arsc
{% endhighlight %}

Dalvik y ART son compatibles, por lo que las aplicaciones para Dalvik funcionan para ART, sin embargo algunas técnicas que funcionan en Dalvik no lo hacen en ART.

ART introduce un compilador antes de tiempo *AOT*, precisamente al momento de instalar y optimizar un *.apk* ART lo traduce a su propio formato ejecutable usando la herramienta dex2oat ubicada en el mismo dispositivo, ésta utilidad acepta un archivo *.dex* y genera un *.oat* (objeto enlazado ELF), el cual aún sigue conteniendo el formato Dex encapsulado en el interior del objeto.

![Alt text](/assets/apk-lifecycle.png)

{% highlight txt %}
| APK                                       | Odex container
| | classes.dex     ====> AOT ======>       |  | Machine code
| | | data section         ^                -----------------
| | classes2.dex           |                | Vdex container
| | | data section       Profile            |  | classes.cdex
                                            |  | shared data section
                                            |  | verification data
{% endhighlight %}
Previo a la implementación de ART, existía únicamente la máquina virtual de Dalvik, sin embargo sustanciales cambios se hicieron debido a que el compilador justo a tiempo *JIT* es lento y demandante en recursos, la recolección de basura *GC_FOR_ALLOC* causaba *jittering*, es de arquitectura de 32-bit y tenía varias debilidades en cuanto a la seguridad por diseño.

Previo a la version 5.0  de android, Dalvik usaba dexopt al momento de instalar un *.apk* y hacer la optimización, volcaba los objetos optimizados *.odex* en */data/dalvik-cache*.

El formato de un archivo DEX es el siguiente: 

![Alt text](/assets/dexfile.PNG){: width="250" }

1. Dex magic header ("dex\n" y version "035")
2. Adler32 of header (offset+12) (checksum)
3. Signature SHA-1 hash of file (20 bytes)
4. Total file size
5. Header size (0x70)
6. Link size (Unused 0x0)
7. Link offset (Unused 0x0)
8. Map offset (Location of file map)
9. Number of String entries
10. Number of Type definition entries
11. Number of prototype (signature) entries
12. Number of field ID entires
13. Number of class definition entries
14. Data (map + res of file )

Elemento del mapa, map_offset.

![Alt text](/assets/hexcode-dexfile.PNG){: width="500" }

Se puede consultar todo el formato del archivo Dex en la [documentacion del sistema Android](https://source.android.com/devices/tech/dalvik/dex-format), en la sección de Runtime estan también los detalles sobre el formato de las instrucciones del bytecode, mejoras, constantes, etcétera.

Para comprender mejor el sistema de lanzamiento de apps, es indispensable entender el contexto a bajo nivel donde ocurre todo el proceso, para ello he incluido éste grafico que complementa los otros ciclos de vida presentados, y entender que cada componente del ciclo de vida en el kernel es un largo proceso por sí mismo.

![Alt text](/assets/android-kernel-lifecycle.png)

El Bootrom es un espacio de memoria no volátil de sólo escritura, normalmente una memoria flash embebida en el chip del procesador, ésta, junto con el mismo CPU, el DSP y el modulo LTE (procesador de señal digital de la banda celular) conforman el corazón del sistema físico de android.

Cuando el procesador enciende, ejecuta el Bootrom, éste dispara la ejecución del Bootloader cuyo trabajo es preparar la ejecución del Kernel, cargando  así los archivos necesarios en memoria y preparando el mismo código del kernel en memoria.

El kernel inicia los drivers, monta el sistema de archivos raíz, prepara el caché, sistemas de protección de memoria *protected memory*, el planificador de tareas "*scheduler*" y dispara el proceso "*init*"

Init es un proceso *root*, que tiene la responsabilidad de montar directorios como /sys, /dev, /proc y correr init.rc que inicializa entre otras variedades los demonios del sistema como el Service Manager, Media Server entre otros, lo cual no lo diferencia mucho de otras distribuciones de Linux.

Lo que varía, sin embargo, en el proceso *init* es que él mismo [dispara un objeto identificado como *app_process* y llama la función *main()* del proceso *X-Zygote*](https://elinux.org/Android_Zygote_Startup).

El proceso *app_process* se encarga de iniciar ART, mientras que la otra función enciende el Zygote, que permite enlazar código entre máquinas virtuales Dalvik/ART, a diferencia de la máquina virtual de Java, donde cada instancia tiene su propia copia de las librerías núcleo y los objetos del *heap*. Zygote inician precargando todas las clases y recursos que una app podría llegar a necesitar en tiempo de ejecución en la memoria del sistema. Después escucha las conexiones en el socket por solicitudes de inicio de las nuevas aplicaciones y cuando recibe una petición de inicialización, se bifurca "*fork*" a sí mismo para lanzar la nueva aplicación. Podría decirse entonces que sirve como padre de todas las aplicaciones Android. 

*(Cuando el ART identifica que un proceso no se esta ocupando y no será necesario, se reclama el espacio de memoria asignado al mismo.)*

La característica de bifurcación viene con la implementación de la técnica de administración de recursos [*copy-on-write*](https://en.wikipedia.org/wiki/Copy-on-write) del Kernel de Linux, es decír que asigna las páginas del nuevo proceso a las del proceso padre y hace copias sólo cuando el nuevo proceso escribe en una página de la memoria, todos los procesos iniciados desde Zygote utilizan su propia copia, una copia de las clases del sistema y los recursos que ya están cargados en la memoria del sistema.

System Server es el primer proceso iniciado por Zygote, después de que inicializa se separa completamente del proceso que lo inició. Y dispara Activity Manager.

Activity Manager por su parte es responsable del proceso de creación de nuevas tareas de *Activity*, mantener el ciclo de vida de Activity, y administrar la pila *stack* o los *stack frames*. Al finalizar la inicialización también ejecuta un Intent para iniciar el Home Launcher.

Después que el *.apk* en cuestión ha sido instalado en el sistema destino y Home Launcher ha inicializado, éste mismo está constantemente escuchando a eventos *onClick*. El evento "click" nos llevaría al disparo del ejecutador de Android, es decír a iniciar la aplicación.

Android asigna un ID (UID) único a cada app, y todas corren sobre su propio proceso. Recordemos pues, que Android usa el kernel de Linux "*en los huesos*", cada proceso tiene asociado su propia máquina virtual, cualquier proceso de ejecución comienza con la primera máquina virtual haciendo una llamada al método principal del proceso orginal de Zygote, el cual precarga todas las clases enlazadas y los recursos a la memoria. 

Entre los recursos que Zygote carga, está el *Software Component Containers*, el cual una vez cargado aguarda la actividad, servicios o proveedores de contenido de la aplicación y el mapeo de éstas clases precargadas es enlazado o compartido cuando la aplicación es disparada, reduciendo así el tiempo de carga.

Hasta éste punto hemos pasado a abstracciones de alto nivel en la implementación, es decír, a partir de aquí el código está siendo ejecutado sobre Zygote, en el siguiente gráfico se adelanta el ciclo de creación de una actividad:

![Alt text](/assets/activity_creation_graph.png)

El punto de entrada de una actividad es Activity Thread, el cual se encarga de manejar la ejecución de la tarea principal en el proceso de la aplicación, que maneja también *broadcasts*.

La máquina virtual ubica el método *public static void main* y lo invoca en Activity Thread. Una vez hecho ésto, procede a llamar al método [Looper.prepareMainLooper](https://developer.android.com/reference/android/os/Looper#prepareMainLooper()), luego establece el [*handler*](https://android.googlesource.com/platform/frameworks/base/+/master/core/java/android/app/ActivityThread.java#1969) con *thread.getHandler()*

{% highlight java %}
public final class ActivityThread extends ClientTransactionHandler {
    class H extends Handler {
        public static final int BIND_APPLICATION = 0
    }
}
{% endhighlight %}

Por último llama al método Looper.loop() en la tarea principal, ésto inicia la ejecución de mensajes en la cola de mensajes. 

El handler en ActivityThread es responsable de manejar todos los mensajes recibidos en su método onHandleMessage. Puede también, reestablecer una actividad, pausarla, comenzarla, unir servicios, manejar recibidores, trimMemory entre otras cosas.

Una vez en el entorno de ejecución de una *Activity* el ciclo lo manejan otros instrumentos, para navegar entre estados del ciclo de vida de la actividad la clase Activity provee un núcleo de seis llamadas:

- onCreate, onStart, onResume, onPause, onStop, onDestroy

![Alt text](/assets/activity_lifecycle.png)
