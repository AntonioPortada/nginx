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
docker run --rm -p 80:80 -p 443:443 -v ./server/conf.d:/etc/nginx/conf.d -v ./server/nginx:/usr/share/nginx --name nginx nginx:stable-alpine3.23
```

## Varios sitios
En la carpeta de configuración voy a copiar el archivo 'default.conf' y lo voy a pegar con el nombre 'apache.conf', en la directiva de `'server_name'` voy a pegar el valor de uno de los `'Virtual Host'` que he creado 'apache.app.test' y en la directiva `'root'` voy a poner el siguiente valor '/usr/share/nginx/apache'. Haré lo mismo con el 'default.conf' con los siguientes datos 'test.domain' y '/usr/share/nginx/foo'. Dentro de las carpetas 'foo' y 'apache' debe haber un archivo llamado 'index.html'.
Después de realizar los pasos anteriores reiniciamos el servidor y al consultar los dominios deberíamos ver diferentes respuestas.

## Página de error
Primero voy a crear en la carpeta `'/usr/share/nginx/foo'` el archivo 'error.html' con el siguiente contenido: '404 No Found Page' y guardamos.
En el archivo 'default.conf' al final de la directiva 'server' voy a agregar la siguiente linea `'error_page 404 /error.html'`, la directiva 'error_page' recibe como parámetros el código de error '404' y déspues el archivo 'error.html' a devolver, dado que en el contexto 'nginx' entiende que ya estamos en la carpeta con la directiva 'root' solo hacemos referencia al archivo.

## Location
La directiva `'location'` sirve para mapear una ruta específica que nosotros quieramos. Ya sea para autenticar usuarios, una redirección, etc...

## Proxy Reversivo (Reverse Proxy)
Nginx va a actuar como un proxy, en el caso de un 'proxy reversivo', se va a encargar de redirigir el tráfico a los servidores (ips, dominios) y devolver la respuesta al cliente que ha realizado la petición.
Ya que todo está siendo realizado usando docker, voy a crear otros dos contenedores con un servidor apache y otro nginx, el contenedor apache será mapeado al puerto 8080 y el nginx al 8081.

``` bash
docker run -dp 8080:80 --name apache httpd && docker run -dp 8081:80 --name nginx-serve nginx
``` 

Después pegamos lo siguiente en el `'default.conf'` antes de la directiva `error_page`:

```
  location /apache {
    proxy_pass http://host.docker.internal:8080/;
  }

  location /nginx {
    proxy_pass http://host.docker.internal:8081/;
  }
``` 

Finalmente reiniciamos nuetro servidor nginx y listo, en el navegador prueba con 'http://nombre_dominio/apache' y 'http://nombre_dominio/nginx', deberías ver las salidas correspondientes a cada ruta. (nombre_dominio es el valor que va después de la directiva 'server_name' en el default.conf)

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

Dado que todo esta ejecutandose desde docker y nos estamos conectando entre contenedores, para que docker sepa como buscar y conectarse debemos usar la ruta `'host.docker.internal'`

# Extra
Debes tener en cuenta que si tienes varias configuraciones, nginx tomará la primera que aparezca, en caso de poner un domain que no exista, ejemplo:
 - tienes dos archivos, 'apache.conf' y 'default.conf', por orden apache va primero y toma esa configuración, entonces deberías ver 'Apache' en el navegador

Si tienes alguna configuración por default y quieres que la respete, editas ese archivo de configuración en la directiva de listen después del puerto agregas 'default_server', guardas y reinicias el servidor otra vez y listo, tienes 'Hola' como respuesta.

En la ruta `'/var/log/nginx/error.log'` se encuentran los logs del servidor.