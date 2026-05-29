# CESE - Sistemas Operativos de Tiempo Real
## Trabajo Práctico N°: 2 - Comunicación de Tareas de FreeRTOS
### Cohorte 26 - Grupo 08
### Responsable de la entrega:
| N° SIU | Apellidos, Nombres    | Fecha | Deadline  |
| :----- | :--------------------- | :------: | :-------: |
| e2620  | Valdez, Boris Cristian | 28/05 | Semana 06 |

# Informe de Actividad 04: Interrupciones y Sincronización


### ¿Qué funciones de la API de FreeRTOS se pueden usar dentro de una rutina de servicio de interrupción (ISR)?
Dentro de una ISR **solo** se pueden utilizar las funciones de la API de FreeRTOS que terminan con el sufijo `FromISR` o `FROM_ISR` (por ejemplo, `xSemaphoreGiveFromISR()`, `xQueueSendFromISR()`). 
Estas funciones están diseñadas específicamente para no bloquear la ejecución (ya que una interrupción no puede pasar al estado *Blocked*) y manejan una variable especial (`pxHigherPriorityTaskWoken`) para indicar si la acción realizada dentro de la ISR despertó a una tarea de mayor prioridad que la que estaba corriendo antes de la interrupción.

### Métodos para delegar el procesamiento de interrupciones a una Tarea
Dado que las ISR deben ser lo más cortas y rápidas posibles, el procesamiento pesado se "delega" a una tarea (Deferred Interrupt Processing). Los métodos son:
* **Semáforos Binarios:** La ISR hace un "Give" del semáforo y una tarea dedicada bloqueada en un "Take" se despierta para procesar el evento.
* **Task Notifications (Notificaciones de Tarea):** Similar al semáforo binario, pero usando `vTaskNotifyGiveFromISR()`. Es un método más rápido y que consume menos memoria RAM.
* **Colas (Queues):** Si la ISR necesita enviar datos además de señalizar el evento.

### ¿Cómo usar una cola para transferir datos dentro y fuera de una rutina de servicio de interrupción?
* **Desde la ISR hacia una Tarea:** Se utiliza `xQueueSendToFrontFromISR()` o `xQueueSendToBackFromISR()`. Si la cola está llena, la función no se bloquea, sino que retorna `errQUEUE_FULL`, y la ISR debe manejar esa pérdida de datos.
* **Desde la ISR recibiendo de una Tarea:** Se utiliza `xQueueReceiveFromISR()`. Si la cola está vacía, retorna inmediatamente indicando que no hay datos.
En ambos casos, se debe evaluar el parámetro `pxHigherPriorityTaskWoken` y, si es verdadero, invocar a `portYIELD_FROM_ISR()` al final de la rutina para forzar un cambio de contexto inmediato.

### ¿Cuál es el modelo de anidamiento de interrupciones disponible en algunas portaciones de FreeRTOS?
FreeRTOS permite el anidamiento de interrupciones (una interrupción puede ser interrumpida por otra más prioritaria) basándose en dos constantes definidas en `FreeRTOSConfig.h`:
1. `configMAX_SYSCALL_INTERRUPT_PRIORITY`: Define la prioridad máxima de interrupción desde la cual se pueden llamar funciones de la API de FreeRTOS.
2. Interrupciones con una prioridad estrictamente **mayor** (numéricamente menor en la arquitectura ARM Cortex-M) a este límite nunca serán retrasadas por el RTOS, pero **tienen estrictamente prohibido** usar cualquier función de la API de FreeRTOS.

## Paso 03: Observaciones del Comportamiento (Interrupciones y Semáforo Binario)

Al eliminar el *polling* y pasar a atender el botón mediante interrupciones externas (EXTI), sincronizándolo con un Semáforo Binario, se observó lo siguiente:

1. **Eliminación del Polling:** La tarea que consultaba continuamente el estado del botón (`task_btn`) ya no es necesaria para leer el pin. El hardware dispara la ISR (`HAL_GPIO_EXTI_Callback`) solo cuando ocurre el evento real (flanco del botón), ahorrando drásticamente recursos de CPU.
2. **Uso de la API FromISR:** Dentro de la ISR se utiliza `xSemaphoreGiveFromISR()` para entregar el token. Esto señala a la tarea encargada del LED que el evento ocurrió, delegando el procesamiento fuera de la interrupción (Deferred Interrupt Processing).
3. **Cambio de Contexto Inmediato:** Gracias al uso de la variable `xHigherPriorityTaskWoken` y la llamada a `portYIELD_FROM_ISR()` al final de la rutina de interrupción, el *Scheduler* fuerza un cambio de contexto inmediato. La tarea bloqueada en `xSemaphoreTake` despierta en el instante en que la ISR termina, logrando una reactividad de hardware en tiempo real.
4. **Efecto de los Rebotes Mecánicos:** Dado que el botón mecánico genera rebotes físicos (múltiples flancos en milisegundos), la interrupción se dispara varias veces casi al instante. Sin embargo, al usar un semáforo binario, el valor máximo es 1. Por lo tanto, el sistema es naturalmente más resistente a acumular pulsaciones falsas en comparación a si se hubiera usado una cola, actuando como un simple gatillo (trigger).
