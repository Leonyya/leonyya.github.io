---
layout: post
title:  "Notas sobre Android Packers (ARMv7 y x86)"
date:   2022-04-12 12:58:29
categories: android
---

Los packers son herramientas escenciales para el crimeware, ya que componen espectro de métodos y técnicas de evasión y ocultación de cualquier ejemplar, cuyas funciones van desde cambiar el hash de un binario, hasta empaquetar un binario cifrado con otro y ejecutarlo de forma dinámica en memoria, antes de ahondar más en el tema, es necesario contextualizar con un repaso sobre el entorno de ejecución de ART y Dalvik

Sin embargo, si se pide al lector que se haga una distinción entre un *packer* y un ofuscador en los ejemplares modernos posiblemente el lector llegue a la conclusión de que no tienen diferencias, esto es porque los packers modernos implementan criterios de ofuscación, pero no debe eliminar el hecho de que son procesos diferentes.

Un ofuscador tomaría el archivo *apk* original, haría modificaciones sobre sus recursos, sus descripciones y el código *smali* y recompilaría el paquete nuevamente a *.apk* una vez hechas las transformaciones. El packer actúa de forma envolvente, suelen generar *stubs* que cargan con el código original encapsulado, así se le hagan modificaciones o no.  

Las implementaciones de packers de alto nivel tales como [Bangcle](https://github.com/woxihuannisja/Bangcle) o [Ijiami](http://www.ijiami.cn/) en su mayoria implementan técnicas como cifrar las cadenas de texto, elegir identificadores aleatorios para métodos, y variables, volver a firmar el binario, cambiar aleatoriamente el archivo *AndroidManifest.xml* sin cambiar la estructura consiguiendo evadir [el analisis N-grama](https://www.youtube.com/watch?v=E_mN90TYnlg) y pueden usar también *dynamic class loading* aprovechando la definición de la clase [*DexClassLoader*](https://developer.android.com/reference/dalvik/system/DexClassLoader) para impedir el análisis estático.

La técnica de carga de clases dinámicas es una característica nativa de Android, y citando la misma documentación:

    A class loader that loads classes from .jar and .apk files containing a 
    classes.dex entry. This can be used to execute code not installed as part
    of an application.

Es común, sobre todo en Bangcle observar técnicas *anti-debug* y *anti-tamper*

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
