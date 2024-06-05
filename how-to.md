# How To
## Dependencias
Para levantar peertube de la forma que planteamos seguiremos mayormente lo
planteado en la siguiente [guia de instalacion con docker](https://docs.joinpeertube.org/install/docker). 
Para esto, y para completar el resto de las tareas pedidas, necesitaremos los
siguientes programas instalados:
- [ Docker ](https://www.docker.com/)
- [ Docker compose ](https://docs.docker.com/compose/)
    - **Nota: dependiendo de la version de docker y docker compose puede que ya
    se instale con docker mismo. Si es asi el comando sera `docker compose` en
    vez de `docker-compose`**
- [ Wireshark ](https://www.wireshark.org/)
- Alguna herramienta de descarga de archivos como puede ser `curl` o `wget`. 
Nosotros vamos a usar wget
 
Ademas de este software deberemos contar con:
- Un dominio(o dos) propio
- Una IP publica donde podamos usar el puerto 80 y 443
    - **Nota: si bien en algunos casos puede que nuestro proveedor nos de una
    IP publica propia para nosotros, no es raro que nuestro router este detras
    de un NAT con lo que incluso si hiciesemos `Port forwarding` desde el mismo
    no tendriamos acceso a estos puertos**

## Antes de comenzar
Previo a la configuracion de `peertube` en si mismo es importante que revisemos
que tengamos la IP publica con un dominio nuestro apuntando a ella y que la 
misma se haya correctamente propagado. Esto se puede realizar a traves de 
paginas como [https://dnschecker.org](https://dnschecker.org). 
Tambien es clave revisar que nuestro servidor `peertube` tenga expuesto el 
puerto `80` y `443`(los puertos de `http` y `https` correspondientemente).
Si se desea tener livestreaming hay que exponer tambien el puerto `1935`

## Configuracion
Una vez alcanzado este momento podemos simplemente seguir la guia que 
previamente mencionada. Para esto vamos a crear una carpeta `peertube` donde
vivira nuestro proyecto. Luego, en una terminal, correremos los siguientes
comandos:
```bash
curl https://raw.githubusercontent.com/chocobozzz/PeerTube/master/support/docker/production/docker-compose.yml > docker-compose.yml

curl https://raw.githubusercontent.com/Chocobozzz/PeerTube/master/support/docker/production/.env > .env

mkdir -p docker-volume/nginx
curl https://raw.githubusercontent.com/Chocobozzz/PeerTube/master/support/nginx/peertube > docker-volume/nginx/peertube
```

Esto nos descargara dos archivos con los que interactuaremos, `.env` y 
`docker-compose.yml`. Tambien nos creara una carpeta `docker-volumes` donde 
se iran guardando distintos archivos de docker. La misma contendra cosas como 
los certificados del dominio, la base de datos, entre otras. En esta carpeta,
dentro de la carpeta hija `nginx` se descargara el archivo de nginx que usara
el webserver que se configura en el `docker-compose.yml`. No es necesario que
interactuemos con este archivo salvo que desiemos hacer cambios sobre el. El
alcance de esto esta fuera del proyecto este.

Volviendo a los archivos de configuracion, deberemos abrir el archivo `.env`. 
En el mismo habra que hacer algunos cambios. Se debe buscar los siguientes
textos y reemplazarlos acorde:
- "\<MY POSTGRES USERNAME\>": Esto puede ser cualquier username siempre que sea
un string continuo(sin espacios, caracteres especiales, etc)
- "\<MY POSTGRES PASSWORD\>": Lo mismo que para username
- "$POSTGRES_USER": Este tiene que ser el mismo string que el username puesto
anteriormente. En algunas versiones de docker este paso no es necesario
- "$POSTGRES_PASSWORD": Lo mismo ocurre aqui con la contraseña
- "\<MY DOMAIN\>": Este se debe completar con el dominio que previamete adquirimos
y que apunta nuestra futura instancia de peertube
- "\<MY EMAIL ADDRESS\>": admin@<MY DOMAIN>
- "\<MY PEERTUBE SECRET\>": Este campo se debe generar utilizando el siguiente
comando: `openssl rand -hex 32`

Con estos pasos completos nos enfocaremos en obtener los certificados de 
dominio para tener comunicacion `HTTPS`. Para lograr esto utilizaremos `certbot`
el cual dejara los certificados en una carpeta especifica donde seran renovados
continuamente. El comando a correr sera el siguiente:
```bash
mkdir -p docker-volume/certbot
docker run -it --rm --name certbot -p 80:80 -v "$(pwd)/docker-volume/certbot/conf:/etc/letsencrypt" certbot/certbot certonly --standalone
```

Al realizar este paso se nos pedira cierta informacion que deberemos completar:
1. un email, no requiere ser el mismo que el de peertube
2. Aceptar los terminos y condiciones
3. (opcionalmente) subscribirnos al newsletter. Se enviara un mail de 
confirmacion de la subscripcion con lo que no es necesario procuparse por el 
spam si nos equivocamos y pusimos que si 
4. el dominio que deseamos autenticar. Este debe ser el mismo dominio que se 
viene utilizando

Con todos estos pasos completos estamos listos para ejecutar nuestra instancia
de peertube por primera vez

## Primera ejecucion
Para ejecutar peertube simplemente debemos hacer:
```bash
docker-compose up
```
**Nota: aca es importante lo que se menciono al principio de la guia. Puede que
haya que usar `docker compose` en vez de `docker-compose`**

Es importante notar en este paso que peertube nos informa de la contraseña de 
root por defecto. Si se desea cambiar, debemos correr en una terminal aparte:
```bash
docker compose exec -u peertube peertube npm run reset-password -- -u root
```
Esto nos pedira que insertemos una nueva contraseña

Si, en cambio, queremos utilizar la contraseña por defecto pero cerramos o 
limpiamos la terminal por algun motivo, podremos hacer:
```bash
docker compose logs peertube | grep -A1 root
```
Esto solo funciona mientras siga el log original de la primera vez que se inicio
y no se haya cambiado la contraseña del usuario root


## Ejecucion
Con todos estos pasos completos estamos listos para ejecutar peertube
```bash
docker-compose up
```
opcionalmente podemos hacer
```bash
docker-compose up -d
```
si deseamos que comience como un daemon y de esta forma no atarlo a una terminal

## Administracion
La primera vez que hacemos login nos va a aparecer un cartel donde nos invita
a configurar nuestra instancia. Se puede simplemente ignorar pero esta bueno
aprovechar este momento para revisar las configuracion y modificar lo deseado.
En cualquier otro momento podremos llegar a este mismo panel desde 
`administracion`

### Sincronizacion de servidores
La sincronizacion de servidores en peertube se hace mediante lo que se llama 
federacion. Para lograr sincronizarte con un servidor hay que seguir los
siguientes pasos:
- Iniciar sesion con un usuario administrador de la instancia
- Ir a `Administration` -> `Federation` -> `Following`
- Presionar el boton de `Follow` e ingresar el `root@` seguido del dominio de
la otra instancia. Peertube dice que se puede utilizar solo el dominio de la
otra instancia pero en nuestra experiencia esto no funciona
- Listo, nuestra instancia deberia estar siguiendo a la otra con lo que pronto
se sincronizaran los videos

### Livestream
El livestream no viene habilitado por defecto en peertube. Para habilitarlo hay
que ir a `administration` -> `configuration` -> `live streaming` -> 
`allow live streaming`.

### Plugins y temas
Desde la pagina de `administracion` podemos tambien agregar plugins/themes 
desde la tab con el mismo nombre. Esto es tan sencillo como buscar el que 
deseamos, presionar instalar y refrescar la pagina una vez termina de instalar
para ver los cambios.

## Analisis de paquetes con Wireshark
El mayor problema que surge aca es que la comunicacion tanto entre servidor y
navegador, como entre servidores se encuentra encriptado ya que se usa `HTTPS`
por lo que tendremos que realizar cierta configuracion ambos en wireshark y en
el navegador. Para esto vamos a realizar los siguientes pasos:
- Cerrar todos los browsers que tengamos abiertos
- En una terminal correr, modificando `$pathAbsoluto` por un path que querramos:
```bash
SSLKEYLOGFILE=$pathAbsoluto/sslkeylog.log google-chrome
o
SSLKEYLOGFILE=$pathAbsoluto/sslkeylog.log firefox
```
  Esto nos abrira un google chrome o firefox que logea las claves generadas en la 
  comunicacion ssl en un archivo `sslkeylog.log` en la carpeta que pusimos
- Abrir `wireshark`
- Ir al menu a `Edit` -> `Preferences` -> `protocols` -> `TLS`(
en versiones viejas puede aparecer como `SSL`)
- Establecer en `(Pre)-Master-Secret log filename` el archivo que seteamos
para la comunicacion encriptada del navegador

De esta forma deberiamos ser capaces de ver los paquetes encriptados entre
chrome y nuestra instancia de peertube

Tambien podemos analizar la interfaz virtual de docker para poder ver paquetes
que el nginx interno a la solucion envia desencriptados al container de 
peertube para poder analizar algunos paquetes extra

Desgraciadamente, es muy dificil acceder a los paquetes salientes de la 
instancia de peertube ya que salen encriptados y no tenemos una forma sencilla,
como en el caso de chrome/firefox de desencriptarlos
