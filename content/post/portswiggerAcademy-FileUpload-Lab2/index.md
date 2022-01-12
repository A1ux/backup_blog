+++
author = "Alux"
title = "Portswigger Academy Learning Path: File Upload Lab 2"
date = "2021-12-26"
description = "Lab: Web shell upload via Content-Type restriction bypass"
tags = [
    "file upload",
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

# Lab: Web shell upload via Content-Type restriction bypass

En este <cite>laboratorio[^1]</cite>la finalidad es subir una shell al servidor para luego poder extraer o recuperar informacion de este. En este caso tenemos que leer el archivo `/home/carlos/secret`

## Reconocimiento

Cuando ingresamos con la cuenta de `wiener:peter` tenemos una opcion para subir archivos, en este caso el avatar del usuario, pero al intentar subir una imagen no nos da una opcion para poder elegir imagen u otro, y tambien se pueden subir archivos `.php`

![Sistema permite la subida de archivos](upload.png)

Y nos muestra que el archivo ha subido correctamente sin problemas aunque sea tipo `PHP`

![Archivo subido al servidor](uploadfile.png)

## Explotacion

Ahora lo que haremos es crear un archivo php con la siguiente funcion, que lo que hara es leer el archivo `secret` de carlos, lo llamaremos shell.php y repetimos el proceso anterior.

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

Cuando lo subimos accedemos al archivo y podemos ver los datos del archivo:

> El archivo se guarda en la ruta /files/avatars/shell.php

![Lectura de archivo secret de carlos](key.png)

Y con esto resolvimos el lab, pudiendo subir un archivo php que ejecuta acciones o comandos en el servidor.

![Laboratorio resuelto](resuelto.png)


[^1]: [Laboratorio](https://portswigger.net/web-security/file-upload/lab-file-upload-web-shell-upload-via-content-type-restriction-bypass)