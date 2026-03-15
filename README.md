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
docker run --rm -p 80:80 -p 443:443 \
-v ./server/conf.d:/etc/nginx/conf.d \
-v ./server/nginx:/usr/share/nginx \
--name nginx nginx:stable-alpine3.23
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
docker run -dp 8080:80 --name apache httpd \
&& docker run -dp 8081:80 --name nginx-serve nginx
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

## Balanceador de cargas
Nginx también sirve como balanceador de cargas, en este ejercicio voy a editar el 'default.conf' para hacer este ejercicio, la directiva nueva a usar es `'upstream'`, después de la directiva se coloca un nombre que será como la variable que contiene los servidores. Cabe mencionar que estamos usando docker para esto, por eso se usa de la siguiente forma.

```
upstream backend {

  server host.docker.internal:8090;
  server host.docker.internal:8091;
  server host.docker.internal:8092;
  server host.docker.internal:8093;

}
```

Después dentro de la directiva 'server' agregar un 'location' con un 'proxy_pass' asignando la "variable" que definimos en el 'upstream' (backend).

```
server {
  listen 80 default_server;

  location / {
    proxy_pass http://backend;
  }
}
```

# Https (certificados)
En está sección agregaré un dominio real y configuraré todo lo necesario para tener 'https' habilitado en mi sitio. Voy a usar la página [duckDNS](https://www.duckdns.org) para generarlos (en el proceso asumiré que ya se tiene el dns).

## Creando certificado
La herramienta que voy a usar para crear los certificados es `'certbot'`.
Dado que por ahora el ejercicio lo estoy realizando con docker, el certificado se debe generar como 'standalone'.
El siguiente comando nos solicitará algunos datos, ingresalos para continuar con el proceso. Al finalizar habrá creado la carpeta `'letsencrypt'`. Se debe correr como root.

```bash
sudo certbot certonly --standalone -d domain.duckdns.org
```

## Agregando certificados
Después de tener los certificados creados los mapeamos al docker y ejecutamos.

```bash
docker run --rm -p 80:80 -p 443:443 \
-v ./server/conf.d:/etc/nginx/conf.d \
-v ./server/nginx:/usr/share/nginx \
-v /etc/letsencrypt/live/domain.duckdns.org/fullchain.pem:/etc/letsencrypt/live/domain.duckdns.org/fullchain.pem:ro \
-v /etc/letsencrypt/live/domain.duckdns.org/privkey.pem:/etc/letsencrypt/live/domain.duckdns.org/privkey.pem:ro \
--name nginx nginx:stable-alpine3.23
```

Para poder visualizar el candado verder y no recibir la alerta de 'página insegura', voy a crear otro sitio en '/etc/nginx/condif.d', para intentar mantener un control, voy a crear un archivo diferente por sitio o rutas.

```bash
vi ssl.conf
```

Después pegamos lo siguiente:

```
server {

  listen 443 ssl;
  server_name tyj-tech;
  
  ssl_certificate /etc/letsencrypt/live/tyj-tech.duckdns.org/fullchain.pem;ssl_certificate_key /etc/letsencrypt/live/tyj-tech.duckdns.org/privkey.pem;
  
  root /usr/share/nginx/ssl;

}
```

Ahora creamos la respuesta del servidor. En la ruta '/usr/share/nginx/ssl' ejecutamos el siguiente comando.

```bash
vi index.html
```

Y por último pegamos esto:

```
This is the https version.
<br/>
Thank you!
```

Finalmente reiniciamos el servidor y veremos que todo funciona bien.
Por ahora tenemos diferentes respuestas al consultar 'http' y 'https'.

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

Antes de consultar el balanceo se deben crear 4 contenedores más para simular los servidores. Con el siguiente comando ejecutas los 4 y como ejercicio voy a cambiar la respuesta de cada uno a servidor 1, servidor 2, etc ...

```bash
docker run --rm -dp 8090:80 --name server_one nginx:stable-alpine3.23 && \
docker run --rm -dp 8091:80 --name server_two nginx:stable-alpine3.23 && \
docker run --rm -dp 8092:80 --name server_tree nginx:stable-alpine3.23 && \
docker run --rm -dp 8093:80 --name server_four nginx:stable-alpine3.23
```

Al agregar el balanceador de cargas se puede dar un peso o carga a un servidor específico. Si nuestro servidor principal tiene más recursos y es más potente; haremos que se cargue más veces para aprovecharlo.

```
server host.docker.internal:8090 weight=4;
```

Con la configuración anterior, cada que actualicemos, entre el orden de los servidores va a salir el servidor uno.
