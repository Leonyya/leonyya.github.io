---
layout: post
title:  "Notas sobre Android Packers (ARMv7 y x86)"
date:   2022-04-12 12:58:29
categories: android
---

Los packers son herramientas escenciales para el crimeware, ya que componen espectro de métodos y técnicas de evasión y ocultación de cualquier ejemplar, cuyas funciones van desde cambiar el hash de un binario, hasta empaquetar un binario cifrado con otro y ejecutarlo de forma dinámica en memoria, antes de ahondar más en el tema, es necesario contextualizar con un repaso sobre el entorno de ejecución de ART y Dalvik [al cual ya se le ha dedicado un articulo](http://leonyya.github.io/2022/android-execution-lifecycle.html)

Sin embargo, si se pide al lector que se haga una distinción entre un *packer* y un ofuscador en los ejemplares modernos posiblemente el lector llegue a la conclusión de que no tienen diferencias, esto es porque los packers modernos implementan criterios de ofuscación, pero no debe eliminar el hecho de que son procesos diferentes.

Un ofuscador tomaría el archivo *apk* original, haría modificaciones sobre sus recursos, sus descripciones y el código *smali* y recompilaría el paquete nuevamente a *.apk* una vez hechas las transformaciones. El packer actúa de forma envolvente, suelen generar *stubs* que cargan con el código original encapsulado, así se le hagan modificaciones o no.  

Las implementaciones de packers de alto nivel tales como [Bangcle](https://github.com/woxihuannisja/Bangcle) o [Ijiami](http://www.ijiami.cn/) en su mayoria implementan técnicas como cifrar las cadenas de texto, elegir identificadores aleatorios para métodos, y variables, volver a firmar el binario, cambiar aleatoriamente el archivo *AndroidManifest.xml* sin cambiar la estructura consiguiendo evadir [el analisis N-grama](https://www.youtube.com/watch?v=E_mN90TYnlg) y pueden usar también *dynamic class loading* aprovechando la definición de la clase [*DexClassLoader*](https://developer.android.com/reference/dalvik/system/DexClassLoader) para impedir el análisis estático.



En una [investigacion realizada por Rowland Yu de Sophos](https://www.virusbulletin.com/uploads/pdf/conference/vb2014/VB2014-Yu.pdf) del 2014, se analizaron los *stubs* generados por tres herramientas apkprotect, [Bangcle](https://dev.bangcle.com/), y Ijiami.

El archivo que se proceso en las tres herramientas tenia la siguiente estructura original:

    res/layout/acitivy_main.xml
    res/menu/main.xml
    AndroidManifest.xml
    resources.arsc
    res/drawable-hdpi/ic_launcher.png
    ...
    classes.dex
    lib/armeabi/libhello-jni.so
    META-INF/MANIFEST.MF
    META-INF/CERT.SF
    META-INF/CERT.RSA

Mientras que, a la hora de desempaquetar el apkfile del *stub* generado, Ijiami devuelve la siguiente estructura:

|Archivo                             |Descripcion                       |
|------------------------------------|----------------------------------|
|res/drawable-xhdpi/ic_launcher.png  |                                  |
|...                                 |                                  |
|resources.arsc                      |                                  |
|classes.dex                         |                                  |
|res/menu/main.xml                   |                                  |
|res/layout/acitivity_main.xml       |                                  |
|lib/armeabi/libexec.so              |ARM JNI libreria enlazada nativa  |
|lib/armeabi/libhello-jni.so         |                                  |
|lib/armeaby/libexecmain.so          |ARM JNI load/unload nativa        |
|META-INF/af.bin                     |Binario sin firma                 |
|META-INF/sdata.bin                  |Firma RSA                         |
|META-INF/signed.bin                 |Binario firmado                   |
|AndroidManifest.xml                 |Configurado para implementar      |
|assets/ijiami.dat                   |APK original                      |

*(Esta tabla fue extraida del whitepaper)*

Podemos observar que el contenido del directorio *META-INF* ha cambiado, y hay dos nuevos objetos enlazados *libexec.so* y *libexecmain.so* y se ha creado el directorio *assets* con un *.dat*.

El resultado del paquete generado por Bangcle devuelve la siguiente estructura:

|File                                                   |Descripcion                          |
|-------------------------------------------------------|-------------------------------------|
|assets/meta-data/manifest.mf                           |APK manifiesto                       |
|assets/meta-data/rsa.pub                               |Firma                                |
|assets/meta-data/rsa.sig                               |Firma real con cert                  |
|...                                                    |                                     |
|assets/bangcle_classes.jar                             |classes.dex original cifrado         |
|assets/bangcleplugin/collector.dex                     |plugin de bangcle recolector de info |
|assets/bangcleplugin/container.dex                     |plugin de implementacion de bangcle  |
|assets/bangcleplugin/dgc                               |relacionado al logfile               |
|assets/com.*                                           |ejecutable ARM y x86                 |
|assets/libsecexe.x86.so                                |x86 shared native library            |
|assets/libsecmain.x86.so                               |x86 native main library              |
|classes.dex                                            |                                     |
|lib/armeabi/libhello-jni.so                            |                                     |
|lib/armeabi/libsecexe.so                               |ARM shared native library            |
|lib/armeabi/libsecmain.so                              |ARM native main library              |
|res/layout/activity_main.xml                           |                                     |
|res/menu/main.xml                                      |                                     |
|resources.arsc                                         |                                     |

(*Esta tabla fue extraida del whitepaper*)

A continuacion, la decompilacion del archivo *classes.dex* enlista las siguientes clases para cada uno de los ejemplares, a la izquierda el original, y los ultimos dos son Ijiami y Bangcle:

![alt](/assets/dex-structure.png)

En el caso de Ijiami, se asegura de especificar dentro de la etiqueta application en el AndroidManifest.xml la propiedad [name](https://developer.android.com/guide/topics/manifest/application-element#nm) la cual referencia a una subclase de Application, cuando el proceso de la aplicacion arranca esta es instanciada primero. Esta edicion al AndroidManifest permite cargar el codigo del packer antes que el codigo del paquete original.

Supongamos pues que el archivo original contenia la siguiente etiqueta:
{% highlight xml %}
<application android:allowBackup="true" android:debuggable="true" 
android:icon="@drawable/ic_launcher" android:label="@string/app_name" 
android:theme="@style/AppTheme">
{% endhighlight %}

El mismo se convertiria entonces en:

{% highlight xml %}
<application android:allowBackup="true" android:debuggable="true" 
android:icon="@drawable/ic_launcher" android:label="@string/app_name" 
android:theme="@style/AppTheme" android:name="com.shell.SuperApplication">
{% endhighlight %}

El punto de entrada de la aplicacion es SuperApplication, el cual hereda la clase Application, carga y ejecuta la clase NativeApplication.

{% highlight java %}
package com.shell;
import android.app.Application;
import android.content.Context;

public class SuperApplication extends Application {
    protected void attachBaseContext(Context paramContext) {
        super.attachBaseContext(paramContext);
        NativeApplication.load(this, "com.xxx.xxx");
    }
    public void onCreate() {
        NativeApplication.run(this, "android.app.Application");
        super.onCreate();
    }
}
{% endhighlight %}

La interfaz de NativeApplication por otro lado carga las librerias nativas que lleva todo el proceso de desempacar, cargar y ejecutar el codigo cifrado.

{% highlight java %}
package com.shell;
import android.app.Application;
public class NativeApplication {
    static {
        System.loadLibrary("exec");
        System.loadLibrary("execmain");
    }

    public static native boolean load(Application paramApplication, String paramstring);
    public static native boolean run(Application paramApplication, String paramstring);
    public static native boolean runAll(Application paramApplication, String paramstring);
}
{% endhighlight %}

En este caso en particular *lib/armeabi/libexec.so* provee las implementaciones para reconocer e interpretar el META-INF, para verificar la firma e integridad de los datos cifrados usando los ciphers RSA y AES, entonces decifra el *.dat* al classes.dex original para posteriormente ejecutar el codigo con *DexClassLoader*.

Sin embargo una de las caracteristicas encontradas en Ijiami en esas fechas era la habilidad de cambiar el dex header original

Sin embargo una de las caracteristicas encontradas en Ijiami en esas fechas era la habilidad de cambiar el dex header original. Dicha modificacion se realizaba al inicio del del archivo hasta 0x28, llenandolo de valores aleatorios. Como resultado el valor DEX_FILE_MAGIC 'dex\n035\0' se veia alterado y por tanto podia detener la traza de la depuracion en tiempo de ejecucion.




Existen por si mismos muchos metodos de ofuscacion que aprovechan el diseno del lenguaje y las interfaces con el runtime, exceptuando pues fallas de seguridad del ART, la siguiente tabla es presentada por los creadores de Obfuscapk en su [whitepaper](https://www.sciencedirect.com/sdfe/reader/pii/S2352711019302791/pdf), la cual enumera las tecnicas de ofuscacion que implementa dicha herramienta.


| Trivial        | Renaming     | Encryption            | Code               | Reflection         |
|----------------|:------------:|:---------------------:|:------------------:|-------------------:|
| RandomManifest | ClassRename  | LibEncryption         | ArithmethicBranch  | Reflection         |
| Rebuild        | FieldRename  | ResStringEncryption   | Reorder            | AdvancedReflection |
| NewAlignment   | MethodRename | AssetEncryption       | CallIndirection    |                    |
| NewSignature   |              | ConstStringEncryption | DebugRemoval       |                    |
|                |              |                       | Goto               |                    |
|                |              |                       | MethodOverload     |                    |
|                |              |                       | Nop                |                    |





Aqui se supone que va un salto de linea grande joer jaja

Bibliografia

[ 1 ] https://www.virusbulletin.com/uploads/pdf/conference/vb2014/VB2014-Yu.pdf

[ 2 ] https://attack.mitre.org/techniques/T1540/

[ 3 ] https://securelist.com/roaming-mantis-part-iv/90332/

[ 4 ] https://cryptax.medium.com/a-native-packer-for-android-moqhao-6362a8412fe1

[ 5 ] https://developer.android.com/reference/dalvik/system/DexClassLoader

[ 6 ] https://docs.microsoft.com/en-us/dotnet/framework/reflection-and-codedom/reflection

[ 7 ] https://www.sciencedirect.com/sdfe/reader/pii/S2352711019302791/pdf
https://developer.android.com/topic/security/dex

https://www.microsoft.com/security/blog/2020/10/08/sophisticated-new-android-malware-marks-the-latest-evolution-of-mobile-ransomware/

https://www.researchgate.net/figure/Copying-procedure-when-the-CopyBinaryFile-method-is-invoked_fig20_308180878

https://www.researchgate.net/figure/Packing-and-unpacking-mechanisms_fig31_308180878

https://www.researchgate.net/figure/Structure-of-an-APK-file-packed-by-Bangcle_fig35_308180878
https://www.virusbulletin.com/uploads/pdf/conference/vb2014/VB2014-Yu.pdf
