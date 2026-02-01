# nginx
Ejercicios de configuración para el servidor nginx 

Voy a usar docker para hacer más rápido y fácil el ejercicio, usando la versión más actual de la imagen. Eventualmente llevaré la práctica a AWS en el servicio de EC2 primero con docker y después con instalaciones en el servidor. Con el siguiente comando el servidor nginx estará listo.

``` bash
docker run --rm -p 80:80 -p 443:443 --name nginx nginx:stable-alpine3.23
```
- El flag '--rm' elimina el contener al cerrar el proceso o detener el contenedor.

Al abrir el navegador en 'localhost' verás la bienvenida de nginx.
