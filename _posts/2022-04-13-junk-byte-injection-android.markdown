---
layout: post
title:  "Junk byte Injection en Android pre-kitkat"
date:   2022-04-12 12:58:29
categories: android
---
Junk byte injection:

En un [articulo](https://web.archive.org/web/20170721234424/http://dexlabs.org/blog/bytecode-obfuscation) de un blog que data del 21 de Julio de 2012 el cual se pudo recuperar del Wayback machine, se aborda una tecnica que aprovecha una brecha de seguridad en el Dalvik VM (previo a Android 4.3 aproximadamente y versiones no lolipop ya que estas implementan ART el cual nunca fue susceptible al metodo).

El verificador de Dalvik es un proceso
https://android.googlesource.com/platform/dalvik/+/kitkat-release/vm/analysis/DexVerify.cpp#66 *linear walk*

https://android.googlesource.com/platform/dalvik/+/kitkat-release/vm/analysis/DexVerify.cpp#705 *explicitly checks*

https://android.googlesource.com/platform/dalvik/+/master/docs/verifier.html *verifier doc* 

https://android-review.googlesource.com/c/platform/dalvik/+/57985 *patch*

CLASS_ISPREVERIFIED flag on the modified classes in the application's dex file, which prevents the dalvik verifier from verifying and rejecting the class. I submitted that change specifically to prevent this obfuscation technique

{% highlight ruby %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}