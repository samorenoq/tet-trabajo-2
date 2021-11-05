# TET Proyecto 2

## Requisitos funcionales

- El sistema debe permitir el registro y/o el login con usuario y contraseña o automáticamente utilizando alguna red social como Google u otros.
- El sistema debe permitir que exista una forma de recuperar la contraseña en caso de ser olvidada.
- El sistema debe permitir la visualización de información sobre los proyectos realizados en la asignatura de Proyecto Integrador.
- El sistema debe tener una interfaz amigable y fácil de entender para los usuarios.
- El sistema debe funcionar con WordPress y allí es donde se debe crear el portal web.
- El sistema debe permitir ampliar la información de cada proyecto publicado.

## Requisitos de calidad

### Privacidad

- El usuario debe poder escoger cuáles de sus datos son públicos y cuáles no. Los datos de los usuarios no deben filtrarse ni enviarse a páginas externas sin el debido consentimiento del usuario.

### Seguridad

- El servicio debe contar con un certificado SSL emitido por una Autoridad de Certificación Verificada.
- Todo el tráfico debe estar encriptado y las contraseñas se deben almacenar de manera segura y con un algoritmo de encripción confiable (como SHA256).
- El usuario debe tener la posibilidad de aumentar la seguridad de su cuenta por medio de "two-factor authentication", recibiendo contraseñas de un solo uso por mensaje de texto a su celular y/o correo electrónico.

### Rendimiento

- La aplicación debe poder responder las peticiones en un tiempo no superior a 2 segundos, independientemente del lugar del mundo desde el que se esté haciendo la petición.

### Escalabilidad

- La solución debe poder aumentar su demanda de manera elástica, sin perder rendimiento. En promedio, debe poder procesar peticiones de 200 usuarios simultáneamente, sin incumplir el requisito de rendimiento establecido anteriormente.

### Disponibilidad

- En promedio, durante el año, el servicio no debe tener un tiempo de baja total superior a 9 horas. Es decir, el servicio debe estar disponible para los usuarios el 99.9% del tiempo.

## Jmeter

Para estas pruebas se utilizó la herramienta BlazeMeter, que permite grabar acciones para luego repetirlas muchas veces, con el fin de probar los tiempos de respuesta y el rendimiento de la solución. En este caso, se usaron 50 usuarios en total, haciendo peticiones de creación de nuevas entradas en la página.
![](https://i.imgur.com/3qECHWm.png)
Los resultados fueron los siguientes (teniendo en cuenta que las peticiones se le hicieron a un servidor minimalista, para no incurrir en potenciales costos en GCP, que posiblemente no pudieran ser cubiertos): 
![](https://i.imgur.com/JHEyZgZ.png)

## Fotos del la solución




## Descripción de la solución

### Diseño

![](https://i.imgur.com/XBWuN5L.jpg)

### VPC Network 1

La idea de crear una VPC es para poder tener una red privada con todos los recursos necesarios para desplegar los nodos de la aplicación y para que éstos puedan comunicarse con direcciones privadas entre sí y con otras VPC Networks que se creen, como la que se creó para la base de datos.

#### Cluster de Kubernetes

Para la replicación y fácil escalabilidad de la solución, ésta se puede desplegar en un cluster de Kubernetes, que se encargaría de gestionar los contenedores que contienen la aplicación. En este caso específico, el cluster se hace regional, para que se despliegue el número de nodos necesario (según la demanda) en cada una de las regiones, ayudando a cumplir el requisito de rendimiento, ya que los nodos están distribuidos geográficamente. Por otro lado, en Kubernetes se hacen varias réplicas de cada contenedor, de tal manera que si alguno falla, automáticamente se instancia otro que reemplaza al que falló, ayudando a cumplir con el requisito de disponibilidad.

### Load Balancer

El balanceador de carga se encargará de distribuir las peticiones entrantes a los nodos gestionados por Kubernetes. La idea es que este balanceador de carga sea elástico, de tal manera que pueda crecer si aumentan las peticiones, ayudando a cumplir con los requisitos de escalabilidad, rendimiento y disponibilidad.

### VPC Network 2

En una segunda red privada aparte, se alojará el servicio de base de datos. Esto permitirá que la VPC de la aplicación pueda comunicarse internamente con la VPC de la base de datos, sin que estas dos estén en la misma red, dando una separación entre la presistencia de datos y la funcionalidad normal de la aplicación y ocultando más la capa de datos.

#### Cloud SQL

Para la base de datos, se usó MySQL, ya que la aplicación funciona con Wordpress. Para crear la base de datos, se usó Google Cloud SQL para montar una instancia de MySQL en la segunda VPC, que se puede comunicar con todos los nodos que están en la primera VPC. Este servicio es completamente versátil y elástico, lo cual es importante para el cumplimiento de los requisitos de escalabilidad, ya que a medida que hay más usuarios y se necesitan más recursos, este servicio de GCP hace la base de datos sea fácilmente escalable para no perder rendimiento. Adicionalmente, cumpliendo con los requisitos de rendimiento y disponibilidad, GCP permite que las instancias de SQL que se creen estén alojadas en múltiples zonas, lo cual favorece la tolerancia a fallos.

### Comunicación entre las VPC Networks (VPC Peering)

VPC Peering es una técnica para que dos VPCs se comuniquen internamente entre sí. En este caso, fue usado para comunicar la VPC que contiene el cluster de Kubernetes con todos los nodos con la VPC que contiene el servicio de base de datos. Esto separa la capa de persistencia del resto de la aplicación y permite que el escalamiento de estas capas, a pesar de ser una aplicación monolítica, se haga por separado. Para ilustrar esto, el siguiente diagrama, tomado de la documentación de GCP, es útil: ![](https://i.imgur.com/a8fcEzS.png)


### Caching

El caching se puede implementar con Redis (que en GCP se implementa como Cloud Memorystore) desde GCP, que se encarga de una parte de la gestión. La idea de esto es disminuir la carga de la base de datos, guardando en memoria algunos atributos poco variables. Según el diagrama provisto por GCP, se vería de esta manera: ![](https://i.imgur.com/N6X0KIX.png)
Como se puede ver, las instancias (que en este caso se encontrarían en la primera VPC) primero pasan por Cloud Memorystore para consultas y después, de ser necesario, irían a la segunda VPC a consultar directamente en la base de datos.

### CDN

Para los dominios de freenom, el proveedor de CDN Cloudflare presenta problemas, entonces para el diseño planteado se usaría el CDN de Google, Google Cloud CDN. Google Cloud CDN permitirá cumplir con los requisitos de rendimiento, específicamente con que los usuarios puedan tener tiempos de respuesta bajos independientemente del lugar del mundo en el que se encuentren, ya que Google Cloud CDN crea copias de la página es diferentes servidores en más de 130 lugares alrededor del mundo.

Tomado de: https://targettrend.com/google-cloud-cdn-for-wordpress/


## Costos de la solución

### Load Balancer
GCP proveé una forma de añadir a nuestra solución un balanceador de carga para controlar la concurrencia entre las instancias. Este, con una condición de direccionamento cuesta un aproximado de USD 18,26 por mes.

### VPC Network 1
#### Cluster de Kubernetes
Cada instancia en creada en GCP, con 10 GB de boot disk, tiene un costo USD 49,32 por mes.
##### Replicación
Para calcular los costos de replicación bastaría con tomar este último valor como referencia, por ejemplo, si se necesitan dos instancias, el valor sería de USD 98,64 por mes.

### VPC Network 2
#### Cloud SQL
El costo total de almacenamiento por 500 GB en una base de datos ubicada en Sao Paulo, para fines de cercanía con Colombia, que serían los principales clientes de la aplicación, sería de USD 17,50 por mes.

### Caching
Teniendo en cuenta que el sistema está diseñado para 20.000 usuarios con un nivel de concurrencia del 10% y un almacenamiento de 500 GB, se calculó que un aproximado de consumo por cada usuario sería de 250 MB. De los 200 usuarios que se espera que se conecten con la aplicación a la vez, este valor de consumo asciende a las 5 GB de almacenamiento, que es el valor que estimamos para el caché de la solución. Este tendría un costo de USD 0,10 por mes

### CDN
Para manejar el contenido se hizo una distrubución por zonas de la siguiente forma.

- Sur América: 230 GB.
- Norte América: 150 GB.
- Asia: 50 GB.
- Europa: 50 GB.
- Oceanía: 20 GB.

El costo para esta distribución es de USD 43,40 por mes.



## Procedimiento para la implementación de la solución

Para la implementación de la solución se usaron diferentes guías, como:
- https://hardikr68.medium.com/guide-to-redis-on-gcp-memorystore-1a851490a15e
- https://cloud.google.com/blog/products/databases/running-redis-on-gcp-four-deployment-scenarios
- https://medium.com/@charurajput/deployment-of-wordpress-on-google-cloud-platform-gcp-with-vpc-and-kubernetes-integration-3f9d50dec191
- https://pchaturvedi19989.medium.com/deploying-wordpress-on-gcp-using-kubernetes-engine-e33c0a7147c1
- https://akanksha77.medium.com/creating-gcp-account-deploying-wordpress-to-launch-web-app-with-load-balancer-and-create-sql-db-be4247ca677
- https://targettrend.com/google-cloud-cdn-for-wordpress/

### Preliminares

- Se debe tener una cuenta en GCP y crear aquí un nuevo proyecto.
- Se deben tener listos el archivo de *nginx.conf* y *docker-compose.yaml* para la implementación de la solución monolítica.
- Se debe crear un Firewall Rule en GCP que permita el ingreso de todas las IPs a todos los puertos (por simplicidad).

### Adquisición del dominio
1. Ir a freenom.com
2. Solicitar el nombre del dominio escribiéndolo en la barra de búsqueda. Por ejemplo: ![](https://i.imgur.com/nHMyIWH.jpg)
3. Verificar la disponibilidad y conseguir el dominio (gratis hasta por 12 meses). ![](https://i.imgur.com/7cTQJYj.png)
4. Una vez se tenga el dominio, ir a la gestión DNS para crear un Registro A que apunte a la dirección IP en la que esté alojada la solución (o a la dirección IP del balanceador de carga, si está implementado) y un Registro CNAME para usar www antes del nombre de la página. ![](https://i.imgur.com/C7cnEpp.png)

Una vez cumplidos estos pasos, ya se puede acceder al dominio via HTTP y éste cargará el servicio que se encuentra en la dirección IP pública proporcionada.

### Adquisición del certificado SSL con Letsencrypt

Para la adquisición del certificado SSL se usó Letsencrypt, siguiendo los siguientes pasos (en este caso, para un servidor Ubuntu 20.04):
1. Acceder al servidor que está corriendo el servicio, por medio de SSH (asegurándose de entrar con un usuario que pueda ejecutar comandos con sudo).
2. Actualizar todo e instalar snapd: `sudo apt update && sudo snap install core && sudo snap refresh core`
3. Instalar Certbot en el servidor: `sudo snap install --classic certbot`
4. Asegurarse de que el puerto 80 esté libre, ya que Certbot usará este puerto para el ACME Challenge para verificar que el servidor efectivamente sea el dueño del dominio. Por este motivo, es fundamental que el paso anterior de adquirir el dominio y agregarle los registros al DNS para apuntar al servidor ya se hayan completado exitosamente.
5. Adquirir dos certificados SSL stand alone para el dominio (con y sin www): `sudo certbot certonly --standalone -d tet-proyecto2-samorenoq.tk -d www.tet-proyecto2-samorenoq.tk`
6. Los certificados quedarán en el servidor en la ruta */etc/letsencrypt/live/tet-proyecto2-samorenoq.tk*. Por este motivo, se debe modificar los archivos *nginx.conf* y *docker-compose.yaml*, para que nginx sepa dónde encontrar el certificado y para que docker-compose mapee el volumen correctamente.
    6.1 **nginx.conf**: agregar las siguientes líneas dentro del bloque *server {...* en el que está configurado HTTPS en el puerto 443: <br>`ssl_certificate /etc/letsencrypt/live/tet-proyecto2-samorenoq.tk/fullchain.pem;`<br>`ssl_certificate_key /etc/letsencrypt/live/tet-proyecto2-samorenoq.tk/privkey.pem;`
    6.2 **docker-compose.yaml**: dentro del servicio que usa *nginx* como imagen, en este caso, 'webserver', agregar un volumen para que añada la ruta con los certificados SSL al contenedor internamente. ![](https://i.imgur.com/QU6gMlY.png)
7. Crear un dhparam corriendo el comando `sudo openssl dhparam -out /etc/letsencrypt/ssl-dhparams.pem 4096`

### VPC Network 1

1. Entrar a la consola de GCP.
2. Ir a la sección de *VCP Networks* y crear una red.
3. Los parámetros de la red fueron los siguientes (los nombres aparecen en uso porque fueron los que ya se habían usado) ![](https://i.imgur.com/s8BxLKf.png)

#### Cluster de Kubernetes

1. Entrar a la consola de GCP.
2. Ir al área de *Kubernetes Engine*
3. Ingresar a la sección de *Clusters*
4. ![](https://i.imgur.com/9Ei6RUv.png)
5. Crear un cluster en modo *GKE Standard*
6. Escoger un nombre para el cluster y seleccionar la opción *Regional*![](https://i.imgur.com/5EKaHZx.png)
7. Ir a la sección de default pool y seleccionar el número de nodos por zona. En este caso, habrá tres zonas y el número de nodos dependerá de la demanda. Sin embargo, también se puede seleccionar la opción para que sea autoescalable, lo cual es ideal para cumplir con el requisito de escalabilidad cuando el número de usuarios aumente ![](https://i.imgur.com/0VIgmzo.png)
8. Ir a la sub-sección de *Nodes* y escoger el hardware óptimo para cada nodo. ![](https://i.imgur.com/UQpQEeX.png)
9. Para finalizar, ir a la sección de Networking y seleccionar la red para el cluster, que en este caso será la VPC Network correspondiente a Wordpress ![](https://i.imgur.com/FXcnkud.png)
10. Click en *Create*
11. Ingresar a la terminal de GCP y correr el siguiente comando para entrar a gestionar el cluster: `gcloud container clusters get-credentials cluster-1 --region us-east1 --project helical-gist-323512`
12. Crear la imagen de Wordpress con el comando `kubectl create deployment wp --image=wordpress`

##### Replicación

Para crear réplicas de los contenedores, correr el siguiente comando `kubectl scale deployment wp --replicas=<num_replicas>`

Este comando inmediatamente crea réplicas de los contenedores y se prepara para ejecutarlas inmediatamente sea necesario.

##### Load Balancer

Para crear un balanceador de carga para los nodos, correr el siguiente comando `kubetcl expose deployment wp --type=LoadBalancer --port=80`. Una vez hecho esto, se puede ver tanto en la terminal como en el área de *Load Balancing* que el balanceador de carga está corriendo ![](https://i.imgur.com/xzT46pa.png) <br>
![](https://i.imgur.com/OrvebfU.png)

### VPC Network 2

Como en la red anterior, el procedimiento es el mismo, cambiando los parámetros específicos de cada VPC, como el nombre, el nombre de la subred y el rango de direcciones IP.

![](https://i.imgur.com/nY5DpU4.png)

#### Cloud SQL

Nota: los siguientes pasos se siguieron tomando en cuenta las instrucciones que se encuentran en https://medium.com/@charurajput/deployment-of-wordpress-on-google-cloud-platform-gcp-with-vpc-and-kubernetes-integration-3f9d50dec191

1. Entrar a la consola de GCP
2. Ir a la sección de *SQL* y hacer click en *Create Instance*![](https://i.imgur.com/64KlTJJ.png)
3. Escoger una base de datos de tipo MySQL.
4. Darle una ID y una contraseña a la instancia y seleccionar la opción *Multiple zones*, que permitirá que la BD tenga mayor tolerancia a fallas. ![](https://i.imgur.com/dJS34RB.png). Las características específicas del HW para la base de datos se describen en mayor detalle en la sección de costos.
5. Asegurarse de que la opción *Private IP* esté activa y seleccionar la VPC de la base de datos creada anteriormente y hacer click en *Create Instance*. ![](https://i.imgur.com/vAPzJpQ.png)
6. Crear un usuario con su respectiva contraseña para la base de datos.
8. Finalmente ingresar los datos de la base de datos creada en la página de configuración de Wordpress en /wp-admin/setup-config.php, para que Wordpress tenga acceso a la base de datos. ![](https://i.imgur.com/JXKAekR.png)

**Opcional**: si se quiere conectar directamente a la base de datos por medio de la consola, se puede correr el siguiente comando: `mysql -h <dirección IP pública del servidor de BD> -u <usuario> -p`

### Comunicación entre las VPC Networks (VPC Peering)

1. Ir a la consola de GCP.
2. Ingresar al área de *VPC networks*
3. Ingresar a la sección de *VPC network peering* <br>![](https://i.imgur.com/J3EnbdP.png)
4. Crear una nueva conexión en *Create Peering Connection*
5. Dar un nombre a la conexión, escoger la VPC Network, el proyecto actual y un nombre para la red. Repetir este paso para cada VPC, tanto para la de Wordpress como para la de la base de datos, para que se puedan comunicar internamente entre sí. ![](https://i.imgur.com/XvpFOT6.png)
En este paso se seleccionan ambas VPC networks para crear una conexión entre ellas. El resultado final, con ambas redes funcionando, debería ser el siguiente: ![](https://i.imgur.com/vFf8D72.png)

### Caching

1. Entrar a la consola de GCP.
2. Ir al área de *Memorystore*
3. Entrar en la sección de Redis
4. Crear instancia <br> ![](https://i.imgur.com/CfVK6rj.png)
5. Darle una id a la instancia y una capacidad (en este caso se pone baja, por costos actuales).
6. Seleccionar la VPC de la base de datos en la red a la que Memorystore se conectará <br> ![](https://i.imgur.com/UXGVZ7W.png)
7. Crear
8. Al final, se debería tener la instancia de Redis corriendo <br> ![](https://i.imgur.com/HPLKbXK.png)


### CDN

Nota: las instrucciones para implementar el CDN en GCP se pueden encontrar en https://targettrend.com/google-cloud-cdn-for-wordpress.

1. Ingresar a la consola de GCP.
2. Ir a la sección de *Network Services* e ingresar a *Cloud CDN* en el panel de la izquierda. ![](https://i.imgur.com/3l90YRp.png)
3. Aquí, agregar un origen, que en este caso será el balanceador de carga. Nota: para esto es necesario tener implementado un balanceador de carga, lo cual ya se hizo cuando se montó el cluster de Kubernetes con los diferentes nodos en pasos anteriores.

Una vez completados estos pasos, ya el CDN de Google debería ser funcional.
