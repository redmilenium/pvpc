# pvpc
Visualizador coste en tiempo real del precio del Kw/h de la electricidad PVPC

Este proyecto nos va a permitir saber el coste de la electricidad PVPC

Foto del prototipo:
![image](https://user-images.githubusercontent.com/48222471/216771040-a1290710-e25a-4d1b-8bd3-cba4f3a47e85.png)

Esquema electrico:
![image](https://user-images.githubusercontent.com/48222471/216772632-8a354635-f90f-4a34-9ace-0c455f1b3070.png)

Los pulsadores nos van a permitir visualizar el precio en cada una de las horas del dia.

Este montaje necesita conexion a internet, pan comido para el wifi del ESP32, y una vez conectado puede obtener los datos referentes al día, hora, mes y año y los  datos referentes al coste de la electricidad PVPC. 

Dichos datos se extraen directamente de REE (Red Electrica Española) en https://api.esios.ree.es/archives/70/download_json?locale=es.

Esto nos permite descargar un archivo JSON del que podemos obtener el coste de la electricidad PVPC.

Los datos se actualizan para el día siguiente a partir de las 20:15. Es decir si nos descargamos el JSON antes de las 20:15 obtenemos los datos del dia en curso y a partir de esa hora, los de mañana.

Para deserializar el archivo JSON he utilizado la web https://arduinojson.org/v6/assistant/#/ que me facilita mucho la parte de programación sobre el JSON.

Es muy facil conseguir los datos referentes al precio de la electricidad en la web con un simple movil, pero este proyecto puede ser la base para, con muy pocos cambios, añadir un rele que solo permita la conexion de un determinado electrodomestico durante las horas de menor coste de la electricidad. Por ejemplo: un termo de agua, un sistema que cargue unas baterias,etc. Esto ya lo dejo en vuestras manos.

En el  soft solo necesitas configurar el SSID y la password y disponer del ide Platformio para grabar el ESP32.






