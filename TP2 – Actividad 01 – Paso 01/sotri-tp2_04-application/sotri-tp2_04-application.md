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