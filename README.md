# Pinned_HTB

## Interceptar trafico en Android

Antes de Android 7.0.0 (API level 24). Android confiaba en certificados instalados por el usuario como cuando instalas el certificado de Burpsuite. Entonces para interceptar trafico en Android modernos tenemos dos opciones: 1 instalar el certificado a nivel de sistema o bien modificar el archivo AndroidManifest para que se confie en certificados instalados por el usuario.

### Instalar certificados en el sistema (necesita root)

Para configurar el Burpsuite se tiene que poner a la escucha el proxy del puerto 8080 en todas las interfaces.


![image](https://github.com/gecr07/Pinned_HTB/assets/63270579/733807f0-e4aa-44f0-99ee-38f5ffb8ed94)


Despues se tiene que exportar el certificado de burpsuite en formato .DER


![image](https://github.com/gecr07/Pinned_HTB/assets/63270579/235a7756-a40b-411c-ad5f-e1e9d5d83c14)


Se tiene que tratar el certificado el primer paso es cambiar de .DER a .PEM para esto se utiliza Openssl y los siguientes comandos:

```
openssl x509 -inform DER -in cacert.der -out cacert.pem
openssl x509 -inform PEM -subject_hash_old -in cacert.pem |head -1
mv cacert.pem <hash>.0
```

#### ¿Que es el subject_hash_old?

En OpenSSL, el subject_hash_old se utiliza para calcular un hash del sujeto del certificado, que luego se utiliza como parte del nombre de archivo para los certificados en el almacén de certificados del sistema. Este hash es crucial para que el sistema operativo o las aplicaciones puedan buscar y verificar rápidamente los certificados.

El sufijo old se usa en versiones más recientes de OpenSSL para diferenciar el método de cálculo del hash del sujeto anterior (legacy) del método nuevo. En versiones más antiguas de OpenSSL (pre 1.0.0), se utilizaba simplemente subject_hash. La versión con old es necesaria debido a cambios en cómo se calcula el hash en versiones más recientes de OpenSSL.

> El subject_hash_old es un valor hash derivado del campo "subject" del certificado X.509. En el contexto de los certificados digitales, el "subject" es una entidad que está siendo identificada por el certificado, generalmente incluye detalles como el nombre de la organización, el país, la unidad organizativa, entre otros.

> Para calcular el subject_hash_old, OpenSSL toma el sujeto del certificado y lo pasa por una función hash (como MD5 o SHA1, dependiendo de la versión y configuración de OpenSSL).


![image](https://github.com/gecr07/Pinned_HTB/assets/63270579/583b7675-1946-4aad-858d-2a66c881f0ce)



## Ruta de Certificados en Android

La ruta donde Android guarda los certificados es:

```
/system/etc/security/cacerts
```

Para continuar tenemos que pasar el certificado .pem a nuestro emulador Android (rooteado). Para poder escribir en esta ruta necesitamos volver a montar el sistema de archivos.

```
adb root
adb remount
adb push <cert>.0 /tmp/
```
En este punto puedes tener acceso con permiso de escritura.  Se movio el certificado a la ruta del sistema donde se guardan los certificados.

```
mv /tmp/<cert>.0 /system/etc/security/cacerts/
chmod 644 /system/etc/security/cacerts/<cert>.0
reboot
```

Hasta este punto Burpsuite ya es capaz de interceptar https. Ahora una cosa es interceptar tradico y otra cosa es el SSL pinning. En este tutorial puedes encontrar mas informacion asi como el otro metodo de "parchear la app".

 
> https://blog.ropnop.com/configuring-burp-suite-with-android-nougat


## SSL Pinning

Se instala el apk arrastrando el archivo al emulador en este caso estamos resolviendo Pinned de HTB un reto del Android path.

![image](https://github.com/gecr07/Pinned_HTB/assets/63270579/ff765159-c28e-4f07-8800-19025e450526)


Esta es una proteccion que se implementa a nivel de app es decir, en este punto ya tenemos el burpsuite interceptando trafico https pero no podemos ver el trafico de la app especifica que queremos interceptar trafico.

Esto se debe a la proteccion SSL Pinning que le dice a las APPs que solo confien en certificados que tienen dentro esta proteccion impide que aunque alguien instale certificados a nivel de sistema( cosa que se hizo arriba) aun asi no se pueda interceptar trafico.


### frida

Por suerte tenemos esta herramienta llamda frida que permite bypassear el ssl pinning ( de nuevo una cosa es interceptar trafico y otra es el ssl pinning). Para usar esta herramienta cliente servidor tenemos que subir el servidor de frida a nuestro Android.


```
adb push .\frida-server-16.0.3-android-x86 /tmp
```

![image](https://github.com/gecr07/Pinned_HTB/assets/63270579/c489b8a8-ad0c-4b68-817a-6b01b229a5fc)


Recuerda ejecutarlo en segundo plano con el &. Para que no se bloquee la terminal. Despues para revisar que si esta funcionando todo utiliza

```
frida-ps -U
```

![image](https://github.com/gecr07/Pinned_HTB/assets/63270579/f109eed2-b686-4459-8c12-8d47ba0d2a20)

Si no se ve una salida similar algo esta fallando y se tiene que revisar. Una vez que todo esta corriendo como debe debes de ejecutar el script de unpinning.js comparto el link.

> https://codeshare.frida.re/@akabe1/frida-multiple-unpinning/


Para poder saber el APP_ID que es el nombre del paquete usa

```
pm list packages | grep -i "Nombre del paquete (apk)"
```


Y para ejcutar frida.

```
frida -U -f <APP_ID> -l frida_multiple_unpinning.js 
```

![image](https://github.com/gecr07/Pinned_HTB/assets/63270579/74529226-d3fa-47ce-bed0-d8d0b34f5cf0)


En este punto ya deberiamos de tener la apk instalada, abierta y lista para que interceptemos los paquetes.

![image](https://github.com/gecr07/Pinned_HTB/assets/63270579/0eae25db-92e3-4bae-b8ec-4c43f78581a2)


Si todo esta funcionando correctamente Burpsuite va a interceptar el trafico sin problemas.


![image](https://github.com/gecr07/Pinned_HTB/assets/63270579/fe15c8cf-41e1-452e-b9ec-5f711d161c05)


FIN.


































