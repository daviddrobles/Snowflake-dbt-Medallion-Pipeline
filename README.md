# DOCUMENTO EXPLICATIVO DEL PROYECTO

## Plataformas utilizadas y contexto

El proyecto ha sido realizado en DBT (Data Build Tool) sobre un .csv que recopilaba información en una ÚNICA TABLA llamada "JUGADORES" acerca de los jugadores de la Bundesliga (Liga Alemana de fútbol) como sus nombres, dorsales, edad, equipos... que se importó a Snowflake donde consideré que la información almacenada en ese fichero era insuficiente y apenas podría crear algo sobre ella.

Debido a esto, decidí crear tablas adicionales generadas de manera aleatoria a partir de la información ya existente, es decir, creé una tabla de clasificación simulando una temporada entera a partir de los equipos recopilados en la tabla "JUGADORES" y la llamé "RESULTADOS", y, además, creé una tabla generada de manera aleatoria a partir de la tabla "RESULTADOS" que llamé "EVENTOS_PARTIDOS" para así poder tener estadísticas a nivel de jugador y equipo.

(En el siguiente punto explicaré cada tabla detalladamente)

Se ha seguido una arquitectura medallion para realizar el proyecto, esto quiere decir que estas tablas nombradas estaban almacenadas en mi capa BRONZE de Snowflake.

Y, por último, el proyecto está COMPLETAMENTE documentado (en DBT) y testeado.

## Tablas principales

Como se ha mencionado anteriormente, el proyecto partió de un .csv original llamado "JUGADORES" que no fue suficiente para la idea que tenía en mente.
Estaban almacenadas en mi capa BRONZE de Snowflake y se importaron a DBT a mi carpeta "STAGING"

### La tabla "JUGADORES" contenía exactamente las siguientes columnas:

- id = El identificador único asignado a cada jugador que diferenciaba a cada uno de ellos.
- name = El nombre conocido de cada jugador.
- full_name = El nombre COMPLETO de cada jugador. ***Esto supuso un problema más adelante***
- age = La edad de cada jugador.
- height = La altura de cada jugador.
- nationality = La nacionalidad de cada jugador.
- place_of_birth = El lugar exacto de nacimiento de cada jugador.
- price = El precio de mercado del jugador.
- max_price = El precio de mercado máximo alcanzado por el jugador.
- position = La posición en el campo de cada jugador.
- shirt_nr = El dorsal de cada jugador.
- foot = La pierna hábil de cada jugador.
- club = El equipo al que pertenece el jugador.
- contract_expires = La fecha de expiración del contrato de cada jugador.
- joined_club = La fecha en la que cada jugador fichó por el equipo.
- player_agent = El agente de cada jugador.
- outfitter = El patrocinador de cada jugador.
- LOAD_AT = ***Campo generado***. La fecha de carga de cada dato. ***Esta columna generada nos servirá más adelante para crear Snapshots***

### La tabla "RESULTADOS" contenía exactamente las siguientes columnas:

- id_partido = El identificador único de cada partido jugado.
- equipo_local = El nombre del equipo que juega como local.
- resultado = El resultado del partido con el siguiente formato (resultado de ejemplo) " 2 - 0 ".
- equipo_visitante = El nombre del equipo que juega como visitante.
- fecha = La fecha en la que se jugó el partido.
- ganador = Indica si el ganador es el equipo local o el visitante o si el partido acabó en empate.
- goles_local = El número de goles marcados por el local. Este campo se generó a partir de la columna de "resultado" separando el resultado en 2 partes.
- goles_visitante = El número de goles marcados por el visitante. Este campo se generó a partir de la columna de "resultado" separando el resultado en 2 partes.
- mvp = El jugador más valioso del partido.

### La tabla "EVENTOS_PARTIDOS" contenía exactamente las siguientes columnas:

- id_evento = El identificador único de cada evento existente en toda la temporada.
- id_partido = El identificador único del partido en el que ocurre el evento.
- equipo = El nombre del equipo al que pertenece el evento.
- id_jugador = El identificador único del jugador que realiza el evento.
- nombre_jugador = El nombre del jugador que realiza el evento.
- tipo_evento = Indica qué evento específico ha ocurrido.
- minuto = El minuto en el que ocurre el evento.

## Snapshot

Antes de nada, *¿qué es una Snapshot?* pues bien, una Snapshot es una funcionalidad utilizada para capturar el histórico de una tabla. Esto nos permite llevar un control de versiones y detectar modificaciones en los registros. 

Y, *¿cómo funciona?* 

DBT compara la tabla de origen con la información ya existente, y si un registro insertado tiene un "id" diferente, se insertará con normalidad, pero si ese "id" ya existe en la tabla y el "LOAD_AT" ha cambiado, se generará una nueva versión de ese registro en la tabla.

*Por ejemplo: El jugador con id "1" ya existe en nuestra tabla desde 2020 y es "David". En 2024 insertamos el jugador con id "2" llamado "Pedro" y volvemos a insertar el jugador con "id" 1 pero con nombre "Pablo".
Cuando DBT compare la información ya existente (el jugador 1) con la información que está siendo ingestada (el jugador 1 y 2), insertará con normalidad al jugador 2, puesto que no existe, pero con el jugador 1 deberá versionarlo ya que ha cambiado su LOAD_AT y deberán existir 2 versiones de ese jugador 1, la que existió (David) desde 2020 hasta 2024 y la que existe actualmente (Pablo) desde 2024.*

Y, *¿para qué?* 

Creé una Snapshot en DBT llamada "jugadores_snapshot_cambios" para poder rastrear los cambios en la tabla de "JUGADORES" a partir del campo "LOAD_AT" considerando que ha habido un cambio cuando ese campo es modificado.
Esta Snapshot me sirvió para estar al tanto y saber si algún jugador ha cumplido años y su edad ha cambiado, si han fichado por otro equipo o si su posición en el campo ha cambiado.

## Macros

Crear macros te simplifica bastante el trabajo y es mucho más fácil realizar algunas tareas complejas.
Por eso creé dos macros:

### ObtenerValores.sql

Esta macro es sencilla, lo único que hace es obtener los valores únicos que existen en una tabla. Realmente lo que ejecuta es un "SELECT DISTINCT" con el fin de obtener todos los registros diferentes existentes.

Esto puede ser útil si queremos saber cuántos registros diferentes hay en una columna, por ejemplo, todos los equipos que existen o todas las nacionalidades diferentes.
A la hora de normalizar tablas, esta macro vendrá bastante bien.

### ObtenerVictorias.sql

La segunda y última macro que utilicé se utilizó para contar el número de veces de victorias locales, visitantes o empates.

## STAGING

La carpeta Staging es la primera carpeta en la que se trabaja con los datos en DBT. 
Se reciben los datos raw para proceder a realizar una primera limpieza con el fin de trabajar con unos datos legibles o correctos.
En esta carpeta se realizan las primeras transformaciones, traducciones y organizaciones.

Sin embargo, existe una carpeta BASE dentro de Staging.

### BASE 

Como he mencionado anteriormente, en la carpeta Staging se empieza con las primeras transformaciones y, en mi caso, comencé a normalizar la tabla de "JUGADORES".
Para facilitar el trabajo y no tener que estar modificando el dato constantemente o renombrando cada columna, utilicé carpeta BASE donde realicé los primeros cambios importantes para, posteriormente, trabajar cómodamente con la tabla ya "arreglada".

**Todos los modelos de BASE tienen como source una tabla existente en Snowflake, a partir de esta, ya se referencian entre ellos en DBT para crear el linaje**

- base_proyecto_final__EVENTOS_PARTIDOS.sql

En esta primera base trabajo sobre la tabla "EVENTOS_PARTIDOS" de Snowflake y lo que realizo son transformaciones a INT (tipo de dato entero) y a varchar (tipo de dato texto) estableciendo los tipos correctos para poder realizar operaciones sobre ellos mas adelante.

- base_proyecto_final__RESULTADOS.sql

En esta segunda base trabajo sobre la tabla "RESULTADOS" de Snowflake y aquí ya realizo una limpieza algo más complicada.

Nuevamente corrijo los tipos de datos, en este caso INT y DATE (fecha).
Y, por otro lado, tenía que introducir el nombre correcto del equipo, así que utilicé un IFF() para modificar su nombre y que esté correctamente escrito.

- base_proyecto_final__JUGADORES.sql

Por último, la tercera base, realizada sobre la tabla "JUGADORES".

En esta base tuve que realizar más operaciones y una limpieza aún mayor, ya que esta tabla era la original del .csv y tenía algunos campos vacíos y tipos de datos que no correspondían.
Volví a corregir los tipos de datos a varchar, date e INT.

Apareció uno de los primeros problemas, muchos campos estaban vacíos, es decir, NULL.
Utilicé una función condicional IFF() para reemplazar los NULL de cada campo por un "Desconocido" o algún otro como "Sin_agente" para evitar errores mas adelante con los campos vacíos.

### RESTO DE MODELOS DE STAGING

Después de haber realizado esos cambios generales en la carpeta BASE para no tener que realizarlos en cada modelo de Staging, normalicé la tabla de JUGADORES generando un modelo para cada equipo, agente, jugador, pierna habil, lugar de nacimiento, nacionalidad, posicion y sponsor.

Referenciando el modelo creado en BASE a partir de la tabla en BRONZE de Snowflake de "JUGADORES" traje toda la información que necesitaba para cada modelo/tabla y generé un identificador único para poder relacionar los modelos entre sí y que cada id exista en todas las tablas que debe aparecer.

Por ejemplo: stg_proyecto_final_equipos.sql > selecciono únicamente la columna "equipo" (en lugar de *) de mi BASE de "JUGADORES" para optimizar la consulta y genero una id única a partir del nombre de cada equipo con {{dbt_utils.generate_surrogate_key(['equipo'])}} as id_equipo

***Nota: utilizar dbt_utils.generate_surrogate_key puede no ser la mejor práctica***

Realicé este mismo proceso en todos mis modelos de Staging.

## INTERMEDIATE 

La carpeta intermediate de DBT es utilizada para obtener campos calculados y operaciones evitando tener que realizarlas más adelante. Esta carpeta no es obligatoria y se utiliza para crear modelos intermedios.

## int_goles_en_contra_equipos.sql

En este modelo intermedio tengo como objetivo calcular los goles en contra que recibe cada equipo, tanto de local como de visitante.
Para ello traigo todo el contenido de las tablas "EQUIPOS" y "RESULTADOS" referenciando mi Staging. 

- Creo una primera CTE con el nombre del equipo, su id y los goles en contra jugando de local uniendo las tablas mediante un join.
- Creo una segunda CTE haciendo lo mismo pero a la inversa, obteniendo los goles en contra jugando de visitante.

***En esta CTE tuve que solucionar un problema. Cuando generé la tabla "RESULTADOS" aleatoriamente, no me generó partidos del FC Bayern Munich actuando de visitante, así que tuve que dividir los datos interpretando que ha jugado la mitad de los partidos como visitante***

- Creé una tercera CTE solucionando este mismo problema con los goles locales y uniendo ambos campos, goles locales y visitantes en una misma "tabla".
- Y, por último, creé una cuarta CTE con el mismo contenido de la anterior y añadiendo una nueva columna que sume los goles locales y visitantes como totales.

## int_stats_equipos.sql

Para este modelo intermedio quería conseguir estadísticas a nivel de equipo como sus partidos como locales, visitantes y totales; victorias como locales, visitantes y totales; goles como locales, visitantes y totales...
Traje todo el contenido de mi tabla "EQUIPOS" y "RESULTADOS" referenciando mi Staging.

Antes de realizar nada, establezco una variable "ganador_" que busque resultados de victorias "Local", "Visitante" o "Empates".

- Creo una primera CTE que cuenta el numero de partidos como local y visitante y utilizo un bucle para crear 3 columnas adicionales que añadan el número de victorias locales, visitantes o empates de cada equipo uniendo las tablas mediante un join.
- Creo una segunda CTE corrigiendo el mismo error anteriormente citado acerca del equipo FC Bayern Munich. Además, cambio el nombre de las columnas, ya que con el bucle utilizado se creaban las columnas con el nombre de "local_amount", "visitante_amount" y "empate_amount", corrijo sus tipos de dato y añado un nuevo campo calculado sumando las victorias locales y visitantes para obtener el número total.
- Creo una tercera CTE añadiendo los goles locales y visitantes de cada equipo y corregí los nombres y tipos de dato de varias columnas
- Creo una cuarta CTE corrigiendo el mismo error anteriormente citado acerca del equipo FC Bayern Munich.
- Y, por último, creo una quinta CTE arreglando algunos tipos de dato y añadiendo el último campo calculado sumando los goles locales y visitantes para obtener los totales

## int_stats_jugador.sql

En este modelo intermedio almacené estadísticas a nivel de jugador (como en el anterior modelo intermedio) como sus goles, su número de mvp's y su minuto promedio de gol.
Traje todo el contenido de mi tabla "JUGADOR", "RESULTADOS" y "EVENTOS_PARTIDOS" referenciando mi Staging.

- Creé una primera CTE modificando algunos tipos de dato, renombrando algunas columnas y obteniendo el número de goles y el minuto promedio de gol de cada jugador uniendo las tablas "JUGADOR" y "EVENTOS_PARTIDOS".
- Creé una segunda CTE uniendo la anterior CTE creada con "RESULTADOS" mediante un join para obtener el número de mvp's de cada jugador.

## int_stats_partidos.sql

Finalmente, en mi último modelo intermedio busco almacenar estadísticas pero ahora a nivel de partidos como el numero de partidos totales, el numero de goles totales, locales y visitantes, el promedio de goles por partido, locales y visitantes, numero de victorias, locales y visitantes y número de empates.
Traje todo el contenido de mi tabla "RESULTADOS" referenciando mi Staging.

Nuevamente, antes de realizar nada, volví a establecer la variable "ganador_" para volver a contar el numero de "Local", "Visitante" y "Empates".

- Creé una primera CTE contando el número de registros como el número de partidos, obtuve los goles locales y visitantes, dividiendo el número de goles totales por el número de partidos totales obtuve el promedio de goles por partido, obtuve el promedio de goles locales y visitantes por partido y obtuve el número de victorias locales, visitantes y empates.
- Creé una segunda CTE corrigiendo los tipos de dato y renombrando las columnas.

# MARTS

La carpeta MARTS, la última de DBT, se utiliza para materializar como tablas las dimensiones y tablas de hecho (aunque no todas las tablas de hecho son tablas y ya, algunas son **incrementales**). En esta carpeta se almacenan los modelos finales diseñados para realizar reportes o dashboards.

## CORE

La carpeta CORE dentro de la carpeta de MARTS es utilizada para todas las dimensiones y tablas de hecho comunes que pueden ser analizadas. 

### DIMENSIONES

Aquí se encuentran, lógicamente, mis dimensiones. La gran mayoría son un "SELECT *" de mi Staging, ya que dejé todas las transformaciones, renombramientos y/u operaciones listos, con lo cual, no tenía que realizar nada más.

Excepto en dos dimensiones:

- La dimensión tiempo (dim_tiempo)

Debido a que no tengo ninguna información acerca de las fechas, tuve que crear una dimensión de tiempo entre la fecha más baja y más alta existentes en mi tabla original.

- La dimensión jugador (dim_jugador)

Aunque pueda parecer una dimensión más que únicamente aporta información de cada jugador, debemos tener en cuenta que debe ser incremental ya que pueden aparecer o desaparecer nuevos jugadores a lo largo de la temporada.

Para lograr esto, tuve que materializar el modelo como "Incremental" en base a la clave única que era el "id_jugador".

### TABLAS DE HECHO

Además de mis dimensiones, creé 5 tablas de hecho. Las tablas de hecho almacenan eventos y contienen métricas cuantificables relacionadas con una o más dimensiones. Habitualmente están cambiando periódicamente.

- fact_resultados.sql

Mi primera tabla de hecho no es más que los resultados de los partidos de la temporada y la creé mediante un "SELECT *" de mi Staging. Considero que es una tabla de hecho porque normalmente los partidos de una temporada se juegan jornada a jornada y la información se va ingestando poco a poco, con lo cual, la tabla de hecho cambiaría cada cierto tiempo. La materialicé como tabla.

- fact_eventos_partidos.sql

Mi segunda tabla de hecho va de la mano con la primera, pues son los eventos de cada partido y también la pude crear mediante un "SELECT *" de mi Staging. Vuelvo a considerar que es una tabla de hecha por el mismo motivo anterior. Materializada como tabla.

- fact_clasificacion.sql

La tercera tabla de hecho que quise crear es la clasificación de la liga. 

Esta tabla de hecho sí tiene algo más de "misterio". Tuve que utilizar mi Stage de "EQUIPOS" y dos de mis modelos intermedios "stats_equipos" y "goles_en_contra_equipos".

***¿Para qué?***

Pues de mi Stage de equipos puedo utilizar el nombre de cada equipo, de mi modelo intermedio de "stats_equipos" obtengo las victorias, derrotas, empates, goles a favor, partidos jugados... y de mi modelo intermedio de "goles_en_contra_equipos" obtengo los goles en contra de cada equipo.

Creé una CTE utilizando los goles totales a favor y en contra para obtener la diferencia de goles uniendo los 3 modelos. 
Multiplicando cada victoria x 3 obtengo los puntos por victoria y sumando el número de empates obtengo los puntos totales de cada equipo.
Y me inventé una columna con una fórmula que utilizaré en la siguiente CTE para ordenar la clasificación llamada "numero". Esta columna lo que hacía era multiplicar los puntos x 100, le sumaba la diferencia de goles, victorias, empates y le restaba las derrotas.

Finalmente, creé la última CTE creando un RANK () OVER particionado por la columna "número" para ordenar los equipos del 1ro al último. Le puse de nombre "puesto" y así logré tener la clasificación correcta.

La materialicé como tabla.

- fact_fichajes.sql

La penúltima tabla de hecho que creé era sobre los fichajes que se producían en la liga.

**Esta tabla de hecho no podía ser materializada como tabla, ya que los fichajes pueden producirse en cualquier momento y pueden aparecer nuevos. Debido a esto, la materialicé como "Incremental".**

Traje toda la información de mi Stage de jugador referenciándolo.

Aunque solo me quedé con el id del jugador, el id del agente, el id del equipo, el precio del jugador y la fecha de fichaje por el equipo.

- fact_contratos.sql

Por último, mi quinta tabla de hecho era sobre los contratos de los jugadores con sus equipos.

**Esta tabla de hecho tampoco podía ser materializada como tabla por la misma razón que la anterior. La materialicé como "Incremental".**

Traje toda la información de mi Stage de jugador referenciándolo.

Me quedé con el id del jugador, el id del equipo por el que había firmado, consideré que la fecha de fichaje por su equipo era la fecha en la que iniciaba el contrato, la fecha de expiración del contrato y creé un nuevo campo booleano para comprobar si el contrato estaba activo (TRUE) o no (FALSE) llamado "activo" comparando si la fecha de hoy es mayor a la fecha de expiración del contrato, así sabía que si la fecha actual había pasado la fecha de expiración, el contrato no estaba vigente.

### DATAMARTS

Para acabar la parte de DBT, creé algunos Datamarts.

***¿Qué son los Datamarts?***

Un datamart es un conjunto de datos diseñado para un área específica de negocio permitiendo un acceso rápido y eficiente a los datos que necesiten los usuarios finales.

Son, digamos, casos prácticos donde se podrían utilizar los datos limpios.

Para este proyecto se me ocurrieron varios ya que tenía prácticamente todos los datos pre-calculados en mi carpeta Intermediate:

- datamart_equipos.sql

En este datamart recopilo información acerca de los equipos uniendo mis dos modelos intermedios de "stats_equipos" y "goles_en_contra_equipos" mediante un join.

- datamart_jugadores.sql

Para este datamart traigo información acerca de los jugadores uniendo mis dos modelos intermedios de "stats_equipos" para coger el número de partidos totales y "stats_jugadores" mediante un join.

- datamart_partidos.sql

Por último, en este datamart almaceno información a nivel de partidos referenciando mi modelo intermedio "stats_partidos".
