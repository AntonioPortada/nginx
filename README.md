# nginx
Ejercicios de configuración para el servidor nginx 

Voy a usar docker para hacer más rápido y fácil el ejercicio, usando la versión más actual de la imagen. Eventualmente llevaré la práctica a AWS en el servicio de EC2 primero con docker y después con instalaciones en el servidor. Con el siguiente comando el servidor nginx estará listo.

``` bash
docker run --rm -p 80:80 -p 443:443 --name nginx nginx:stable-alpine3.23
```
- El flag `'--rm'` elimina el contener al cerrar el proceso o detener el contenedor.

Al abrir el navegador en `'localhost'` verás la bienvenida de nginx.

## Archivos de configuración
Las carpetas principales de configuracion están en `'/etc/nginx/conf.d'` los sitios que nginx va a proveer. Y `'/usr/share/nginx'` la respuesta a esos sitios, el html que va a devolver. Un ejemplo es la bienvenida de nginx.
Para los ejercicios voy a mapear estás carpetas a los paths para tener persistencia de las configuraciones.

En la raiz del proyecto voy a crear las siguientes carpetas `'server/conf.d'` y `'server/nginx'`, dentro de conf.d estará el archivo `'default.conf'` y en la carpeta nginx estará `'html/index.html'` que previamente copie del contenedor original. 

Después volvemos a ejecutar el contenedor con volumenes asignados a esas ruta y validamos que siga funcionando.

``` bash
docker run --rm -p 80:80 -p 443:443 -v ./conf.d:/etc/nginx/conf.d -v ./nginx:/usr/share nginx --name nginx nginx:stable-alpine3.23
```

## Varios sitios
En la carpeta de configuración voy a copiar el archivo 'default.conf' y lo voy a pegar con el nombre 'apache.conf', en la directiva de `'server_name'` voy a pegar el valor de uno de los `'Virtual Host'` que he creado 'apache.app.test' y en la directiva `'root'` voy a poner el siguiente valor '/usr/share/nginx/apache'. Haré lo mismo con el 'default.conf' con los siguientes datos 'test.domain' y '/usr/share/nginx/foo'. Dentro de las carpetas 'foo' y 'apache' debe haber un archivo llamado 'index.html'.
Después de realizar los pasos anteriores reiniciamos el servidor y al consultar los dominios deberíamos ver diferentes respuestas.

# Tips
Para simular el ejercicio visitando una URL como 'web.test' o 'app.test' como dominio, en el archivo `'/etc/hosts'` en una mac, al final del archivo agregamos los siguientes datos:

``` 
ip_local    web.test
ip_local    app.test
``` 

después de esa configuración en el archivo `default.conf` agregamos uno de los dominios creados después de la directiva `server_name`, guardamos y reiniciamos el servidor con el siguiente comando:

```
nginx -s reload
```

al visital el `Virtual Host` que has creado, deberías seguir obteniendo la misma respuesta.
