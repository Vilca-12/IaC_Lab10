Una vez ejecutado docker compose up -d, se verificó que todos los servicios estuvieran funcionando correctamente. Para ello, se revisó el estado del backend, frontend, Grafana, Prometheus y los demás contenedores involucrados en la práctica.
Para comenzar con la segunda parte, se ingresó al frontend y se utilizó la opción "Saludar". Cada saludo generado queda registrado en el sistema y posteriormente puede ser monitoreado mediante las herramientas de observabilidad.
Luego se accedió a Grafana y se verificó la conexión de las fuentes de datos. Para ello, se ingresó a Data Sources y se revisaron las configuraciones de Prometheus y Loki. Dentro de cada una se utilizó la opción Test para comprobar que la conexión estuviera funcionando correctamente.
Después de validar las fuentes de datos, se procedió a crear un dashboard. Para ello se ingresó a Dashboards, luego a New y finalmente a New Dashboard. En la pantalla principal apareció la opción Add Visualization, donde se seleccionó la fuente de datos correspondiente para cada panel.

Paso 4: Construcción del Dashboard
CPU del contenedor backend
Se creó un panel utilizando la fuente de datos Prometheus y la siguiente consulta:

sum(rate(container_cpu_usage_seconds_total{name="lab-backend"}[1m])) * 100
Esta consulta permite visualizar el porcentaje de CPU consumido por el contenedor del backend. Se seleccionó la visualización Time Series, se configuró la unidad como Percent (0–100) y se asignó el nombre "CPU contenedor backend (%)". Además, se agregó un umbral de 50% para identificar visualmente cuándo el consumo supera dicho valor.

CPU del host
A continuación, se agregó un segundo panel utilizando nuevamente Prometheus con la siguiente consulta:

100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)
Esta métrica muestra el uso general de CPU de la máquina donde se ejecutan los contenedores. Se configuró como Time Series, con unidad Percent (0–100) y el título "CPU del host (%)".

Logs de aplicación
Posteriormente, se creó un panel de tipo Logs utilizando la fuente de datos Loki. Se utilizó la siguiente consulta:

{tier="application"} | json
Con esta consulta se muestran los registros generados por el backend y el frontend. En caso de requerir únicamente los errores, se puede emplear la siguiente variante:

{tier="application"} | json | level="ERROR"
El panel fue nombrado "Logs de aplicación (API + frontend)".

Logs de infraestructura
Finalmente, se agregó un cuarto panel utilizando Loki con la consulta:

{tier="infrastructure"}
Este panel permite visualizar los logs generados por los componentes de infraestructura como Prometheus, Grafana, Loki y los exporters. Se le asignó el nombre "Logs de infraestructura".
Una vez creados los cuatro paneles, se guardó el dashboard para contar con una vista centralizada de las métricas y logs del sistema.

Paso 5: Configuración de la alarma de CPU
Con el dashboard terminado, se procedió a configurar una alerta en Grafana. Para ello se ingresó a Alerting → Alert Rules → New Alert Rule y se creó una regla llamada "CPU backend > 50%".
Como consulta se utilizó:

sum(rate(container_cpu_usage_seconds_total{name="lab-backend"}[1m])) * 100
Grafana generó automáticamente las expresiones Reduce y Threshold. En el umbral se configuró la condición IS ABOVE 50, indicando que la alerta debe activarse cuando el consumo de CPU del backend supere el 50%.
Posteriormente se configuró un Evaluation Group con un intervalo de evaluación de 10 segundos y un Pending Period de 30 segundos. Esto permite evitar alertas provocadas por incrementos momentáneos del consumo.
También se agregó la etiqueta:

severity = warning
Finalmente, se seleccionó el contact point por defecto y se guardó la regla.

Paso 6: Prueba de la alarma
Para comprobar el funcionamiento de la alerta, se regresó al frontend y se presionó el botón "Generar carga de CPU (30s)" el cual yo lo oprimi unas 3 veces.
Como alternativa, la carga también puede generarse desde la terminal mediante:

curl "http://localhost:3001/load?seconds=60"
Al ejecutarse la carga, se observó en el panel CPU contenedor backend (%) cómo el consumo aumentaba progresivamente hasta superar el umbral configurado.
Luego, al ingresar a Alerting → Alert Rules, se pudo observar el cambio de estados de la alerta:

Normal → Pending → Firing
Una vez concluida la carga, el uso de CPU disminuyó y la alerta regresó automáticamente al estado Normal.
Como evidencia de la práctica, se tomaron capturas del panel mostrando el uso de CPU superior al 50% y del estado Firing de la alerta.

Paso 7: Integración de alarma y logs
Para completar el flujo de observabilidad, se configuró un Contact Point de tipo Webhook apuntando a la siguiente dirección:

http://backend:3001/alerts
De esta manera, cada vez que la alerta se activa, Grafana envía una notificación al backend. El backend registra dicho evento como un log, el cual puede visualizarse posteriormente en el panel Logs de infraestructura.
Con esta configuración se logra observar el ciclo completo de monitoreo:

Métrica → Alarma → Log → Dashboard
Es decir, cuando una métrica supera un umbral establecido, se genera una alerta; la alerta produce un registro en los logs y dicho registro puede ser visualizado desde Grafana, permitiendo un seguimiento completo de los eventos del sistema.

---

### Realice estos cambios (Solución de problemas en producción)

Durante el despliegue real del laboratorio se identificaron problemas de configuración en el entorno. Se aplicaron los siguientes cambios técnicos breves para corregirlos y garantizar el funcionamiento del flujo:

1. **Métrica de CPU Alternativa (Paso 5):**
   * **Cambio:** Se reemplazó la consulta original de CPU por `rate(backend_process_cpu_seconds_total{job="backend"}[1m]) * 100`.
   * **Por qué:** La métrica propuesta en la guía no cargaba datos ni generaba gráficas en Prometheus bajo este entorno. Al utilizar esta métrica nativa obtenida directamente del proceso Node, la gráfica cobró vida y la alarma pudo activarse correctamente, cumpliendo a cabalidad con la función de monitorear y analizar el comportamiento aislado del backend.

2. **Vinculación del Punto de Contacto (Paso 5):**
   * **Cambio:** Se asignó y guardó explícitamente el receptor (`backend-webhook`) dentro de la regla de alerta.
   * **Por qué:** Por defecto la regla apuntaba a un receptor vacío (`empty`), lo que provocaba que Grafana detectara la subida de CPU pero no enviara la notificación a ningún lado.

3. **Validación en el Panel Correcto (Paso 4):**
   * **Cambio:** Se monitoreó el evento de enrutamiento utilizando la consulta `{tier="infrastructure"}` en el panel de soporte.
   * **Por qué:** Como el motor de alertas de Grafana pertenece a los componentes del sistema base, el log de envío del webhook quedó registrado dentro de la infraestructura. Esto permitió confirmar la salida del disparo técnico con la línea de trazabilidad de `ngalert`.

4. **Corrección de la URL del Webhook (Paso 7):**
   * **Cambio:** Se cambió el host del Contact Point de `http://backend:3001/alerts` a `http://lab-backend:3001/alerts`.
   * **Por qué:** Grafana arrojaba errores de conexión fallida al intentar enviar la alerta. Al cambiarlo por el nombre DNS real del contenedor dentro de la red interna de Docker (`lab-backend`), los servicios se comunicaron con total éxito de forma inmediata.



Respuesta a las preguntas 

1. ¿Por qué necesitamos Loki además de Prometheus si ya tenemos /metrics?
Porque ambos se complementan ,Prometheus almacena métricas y nos muestra qué ocurrió y cuándo ocurrió, mientras que Loki almacena logs y permite conocer la causa del problema. 
Por ejemplo, Prometheus puede mostrar un aumento de CPU y Loki puede indicar el error que lo provocó.
2. ¿Qué ventaja aporta que las fuentes de datos de Grafana estén aprovisionadas como código y no creadas a mano?
Una ventaja es que toda la configuración queda automatizada y puede reproducirse fácilmente en cualquier entorno, en caso se necesite volver a levantar el proyecto o migrarlo a otra máquina, las fuentes de datos se configuran automáticamente sin tener que hacerlo manualmente en Grafana.
Además, al estar guardadas en archivos de configuración, pueden almacenarse en Git para llevar un control de cambios. También se reducen errores de configuración que podrían ocurrir al realizar el proceso manualmente.

3. El panel "CPU contenedor" y el panel "CPU host" pueden mostrar valores muy distintos. ¿Por qué? ¿Cuál usarías para alertar sobre una aplicación concreta?
Pueden mostrar valores distintos porque la CPU del host mide el consumo total de la máquina, mientras que la CPU del contenedor solo mide el consumo de una aplicación específica. Para alertar sobre una aplicación concreta utilizaría la CPU del contenedor, ya que muestra directamente el comportamiento de esa aplicación y evita alertas causadas por otros procesos del sistema.
4. ¿Qué diferencia hay entre el evaluation interval y el pending period de una alarma?
El evaluation interval indica cada cuánto tiempo Grafana revisa si se cumple la condición de la alerta. En esta práctica se configuró cada 10 segundos.
El pending period indica cuánto tiempo debe mantenerse la condición antes de activar la alerta. En nuestro caso fue de 30 segundos.

