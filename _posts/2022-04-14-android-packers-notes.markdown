---
layout: post
title:  "Notas sobre Android Packers (ARMv7 y x86)"
date:   2022-04-12 12:58:29
categories: android
---

Las implementaciones de packers y ofuscadores de alto nivel tales como [Bangcle](https://github.com/woxihuannisja/Bangcle), ProGuard o DexGuard en su mayoria implementan tecnicas como cifrar las cadenas de texto, elegir identificadores aleatorios para metodos, y variables, volver a firmar el binario, cambiar aleatoriamente el archivo *AndroidManifest.xml* sin cambiar la estructura consiguiendo evadir [el analisis N-grama](https://www.youtube.com/watch?v=E_mN90TYnlg) y pueden usar también *dynamic class loading (DexLoader classes)* para impedir el análisis estático.



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