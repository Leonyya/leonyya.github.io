---
layout: post
title:  "Notas sobre packers en malware basado en Xamarin."
date:   2014-10-18 12:58:29
categories: android
---

The next table is presented by the creators of Obfuscapk in it's whitepaper [7]:


| Trivial        | Renaming     | Encryption            | Code               | Reflection         |
|----------------|--------------|-----------------------|--------------------|--------------------|
| RandomManifest | ClassRename  | LibEncryption         | ArithmethicBranch  | Reflection         |
| Rebuild        | FieldRename  | ResStringEncryption   | Reorder            | AdvancedReflection |
| NewAlignment   | MethodRename | AssetEncryption       | CallIndirection    |                    |
| NewSignature   |              | ConstStringEncryption | DebugRemoval       |                    |
|                |              |                       | Goto               |                    |
|                |              |                       | MethodOverload     |                    |
|                |              |                       | Nop                |                    |

Bibliografia 
[ 1 ] https://www.virusbulletin.com/uploads/pdf/conference/vb2014/VB2014-Yu.pdf
[ 2 ] https://attack.mitre.org/techniques/T1540/
[ 3 ] https://securelist.com/roaming-mantis-part-iv/90332/
[ 4 ] https://cryptax.medium.com/a-native-packer-for-android-moqhao-6362a8412fe1
[ 5 ] https://developer.android.com/reference/dalvik/system/DexClassLoader
[ 6 ] https://docs.microsoft.com/en-us/dotnet/framework/reflection-and-codedom/reflection
[ 7 ] https://www.sciencedirect.com/sdfe/reader/pii/S2352711019302791/pdf


