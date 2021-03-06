# Despliegue de sistemas predictivos - Práctico 2
> Diplodatos 2019

En esta segunda iteración, vamos a continuar trabajando en nuestra API para
predicción de sentimientos en oraciones, profundizando e implementando lo
siguiente:

1. Testing de stress con *Locust*
2. Monitoreo con *EKL*
3. Scaling de servicios
4. Obtener y almacenar feedback de usuarios
5. Usar traefik como un DNS resolver y descubrir sus features

## 1. Testing de stress con *Locust*

Para esta tarea deberá modificar el archivo `locustfile.py` en la carpeta
`stress_test`.

- Cree al menos un test de uso del endpoint `index` de nuestra API para evaluar
  su comportamiento frente a un gran número de usuarios.
  
  ```python
  @task(1)
  def index(self):
      self.client.get("/")
  ```
  
- Cree al menos un test de uso del endpoint `predict` de nuestra API para
  evaluar su comportamiento frente a un gran número de usuarios.

  ```python
  @task(3)
  def predict(self):
      self.client.post("/predict", params={"text": self.POS_SENT})
  ```

- Detalle en un reporte las especificaciones de hardware del servidor donde está
  desplegado su servicio y compare los resultados obtenidos para diferentes
  números de usuarios.
  
  * Espeficaciones 
    Loctus funciona en un contenedor de docker su declaracion se encuentra en
    [docker-compose](./docker-compose.yml). Las espeficicaciones de la maquina
    donde corre todo el sistema es:
  
      * Procesador Intel(R) Core(TM) i7-7500U, 4 threads
      * Memoria 8 GB
  * Resultados
    El test se realiza con 200 usuarios con una tasa de crecimiento de 1 usuario
    
    <img 
    src="./doc/loctus_test_bar.png" 
    width=90% 
    />
    
    En general se puede responder entre 4 y 5 respuestas por segundo
    
    <img 
    src="./doc/test_01.png" 
    width=90% 
    />
    
    Podemos observar que no se tiene errores por consulta pero si el tiempo de
    respuesta es muy alto. 
    
    Esto se debe al cuello de botella del middleware, ya que se espera que modelo
    termine de procesar para dar una respuesta.
    
    <img 
    src="./doc/test_02.png" 
    width=90% 
    />
    
    
    <img 
    src="./doc/test_03.png" 
    width=90% 
    />


- (Opcional) Reemplace el actual procesamiento de datos secuencial en
  `ml_service.py` por procesamiento en batches. ¿Nota alguna mejora?

## 2. Monitoreo con *EKL*

Si bien la propuesta es ElasticSearch Kibana y Logstash para mejor entendimiento
usaremos tecnologías similares. En este punto se deberá instanciar un stack
compuesto de los siguientes servicios:
  - *mongodb*: Para dar soporte de base de datos a graylog en la gestión de
    usuarios y configuraciones
  - *elasticsearch*: Para el almacenamiento persistente de los datos a ser
    procesados. En este caso serán logs de salida de los contenedores.
  - *graylog*: Responsable de la gestión de elasticsearch. Creará nuestros
    indices rotativos con persistencia configurable y además nos permitirá hacer
    preproceso en la ingestión de datos. Su driver de ingestión de datos nos
    permitirá conectar la salida de los contenedores de manera nativa.
  - *grafana*: Herramienta que utilizaremos para la creación de dashboards para
    visualización. Elegida por su versatilidad y fácil entendimiento.

<img 
    src="./doc/lab_2.png" 
    width=90% 
/>

Graylog

<img 
      src="./doc/lab2_res_1.png" 
      width=90% 
/>

<img 
      src="./doc/lab2_res_2.png" 
      width=90% 
/>


Una vez realizadas las configuraciones iniciales para completar la tarea será
necesario enviar los logs al stack instanciado y visualizar dashboards de
actividad donde se pueda ver en tiempo real las siguientes estadísticas.

- *req/min* que está recibiendo nuestra API
  <img 
      src="./doc/lab2_res_graf_1.png" 
      width=90% 
  />
- *histograma de actividad* diferenciando cuales dieron respuesta positiva y
  cuales negativa.
  
  <img 
      src="./doc/lab2_res_graf_2.png" 
      width=90% 
  />
- *alerta de errores* al recibir más de 10 request con codigo de error (>=400)
  en un minuto
  
  Agregando el siguiente comando para imprimir los codigos de respuesta
  
  ```yml
  api:
    image: flask_api
    container_name: ml_api
    build:
      context: ./api
    command: gunicorn --workers=8 --bind 0.0.0.0:5000 --access-logformat "{\"response_code\":%(s)s}"  --log-level=debug --access-logfile - app:app
  ```
  
  Creacion de Query

   <img 
      src="./doc/alarm_02.png" 
      width=90% 
  /> 
  
  Creacion de alarma
  
     <img 
      src="./doc/alarm_03.png" 
      width=90% 
  /> 
  
  Resultados
  
     <img 
      src="./doc/alarm_01.png" 
      width=90% 
  /> 
  
*AYUDA*: Aqui (https://docs.graylog.org/en/3.1/pages/installation/docker.html)
encontrarán la informacion al respecto de como levantar el stack propuesto. Los
pasos para la configuración inicial serán explicados en el teórico. Ademas
deberán agregar los prints necesarios para poder ingestar los datos minimos que
necesitan para su dashboard.

## 3. Scaling de servicios

El objetivo aqui es duplicar nuestra capacidad de respuesta instanciando un
"worker" más en nuestra infraestructura. Para ello deberíamos aumentar la
cantidad de réplicas que tenemos del contenedor *model* y visualizar las mejoras
en grafana usando nuestro cliente locust para exigir la carga. Para ello deberán
utilizar el comando `docker-compose scale <SERVICE>=<#INSTANCES>`

#### Especificaciones de la Maquina
 * Intel(R) Core(TM) i7-6700 CPU @ 3.40GHz
 * 8 Threads
 * 16 GB RAM

Las pruebas fueron usando locust 200 usuarios con una tasa de crecimiento de 1

### Resultados worker = 1

#### Locust

<img 
      src="./doc/replica_01_locust.png" 
      width=90% 
  /> 


#### Grafana

<img 
      src="./doc/replica_01_grafana.png" 
      width=90% 
  /> 


### Rsultados worker = 2

Comando utilizado

```sh
docker-compose -f docker-compose.yml up --scale model=2
```
#### Locust

<img 
      src="./doc/replica_02_locust.png" 
      width=90% 
  /> 

Se puede apreciar que aumentando el modelo a 2 workers se tiene un mayor numero
de respuestas por segundo. Aumentando el desemepeño del sistema.

#### Grafana

<img 
      src="./doc/replica_02_grafana.png" 
      width=90% 
  /> 

Tambien se observa las mejoras usando grafana. Hay que tener en consideracion
que las mejoras se pueden observar si se dispone de recursos (ejm RAM) ya que
inicialemnte el escalamiento fue probado en un PC con 8 GB. Y el rendimiento en
lugar de mejorar, eran peores (las respuestas por segundo oscilaban en un 1.3).


## (Opcional) 4. Obtener y almacenar feedback de usuarios
En las views de nuestro proyecto deberán completar el endpoint para feedback y
permitir al usuario así acusar una respuesta incorrecta. Almacenar en un csv
todos estos reportes para una futura retroalimentación.

## (Opcional) 5. Usar traefik como un DNS resolver y descubrir sus features
Entre las muchas funcionalidades que traefik tiene integradas podemos encontrar
un balanceador de carga con resoluciones DNS. El desafío propuesto es poder
utilizar traefik y descubrir sus funcionalidades integrandolo en nuestro
proyecto. Las tareas a realizarse son:

- Descargar e instanciar mediante docker run la imagen
  [containous/whoami](https://hub.docker.com/r/containous/whoami) y entender la
  información que esta nos brinda. Dentro de esa información identificar el
  valor que nos permita conocer la IP del contenedor. Esto sera util para más
  adelante.
- Levantar al menos 2 servicios (uno con nuestra API y otro sirviendo la imagen
  pública containous/whoami) y visualizarlos en el dashboard de traefik.
  Verificar también el acceso mediante el DNS http://my.own.api.localhost y
  http://my.own.whoami.localhost respectivamente.
- Una vez desplegados los dos servicios, generar réplicas para nuestro servicio
  whoami con el comando `docker-compose scale whoami=3` y acceder a
  http://my.own.whoami.localhost para verificar el balanceo de carga entre los
  distintos contenedores notando las diferentes IP's de contenedores que reciban
  nuestra petición.

*AYUDA*: Aqui
(https://docs.traefik.io/user-guides/docker-compose/basic-example/) encontrarán
un ejemplo sencillo de uso de traefik incluso con la imagen containous/whoami.
