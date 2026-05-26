# CESE - Sistemas Operativos de Tiempo Real
## Trabajo Práctico N°: 2 - Comunicación de Tareas de FreeRTOS
### Cohorte 26 - Grupo 08
### Responsable de la entrega:
| N° SIU | Apellidos, Nombres    | Fecha | Deadline  |
| :----- | :--------------------- | :------: | :-------: |
| e2620  | Valdez, Boris Cristian | 28/05 | Semana 06 |


## Actividad 02 - Paso 02: Gestión y Funcionamiento de Colas (Queues) en FreeRTOS

Este documento detalla los aspectos teóricos y prácticos del uso de Colas (Queues) en el Kernel de FreeRTOS, consolidados a partir de la experiencia de depuración en la plataforma STM32.

---

## 1. Creación y Eliminación de Colas

### ¿Cómo crear una Cola?
Para crear una cola en FreeRTOS se utiliza la función `xQueueCreate()`. Esta función reserva dinámicamente la memoria RAM necesaria en el *Heap* del sistema para la estructura de control de la cola y el almacenamiento de sus elementos.

* **Sintaxis básica:**
    ```c
    QueueHandle_t xQueue;
    xQueue = xQueueCreate(uxQueueLength, uxItemSize);
    ```
* **Parámetros:**
    * `uxQueueLength`: El número máximo de elementos que la cola puede albergar simultáneamente.
    * `uxItemSize`: El tamaño en bytes de cada elemento (se recomienda usar el operador `sizeof()`).
* **Retorno:** Devuelve un manejador del tipo `QueueHandle_t`. Si no hay suficiente memoria en el *Heap*, retorna `NULL`.

### ¿Cómo eliminar una Cola?
Para liberar los recursos asociados a una cola cuando ya no se requiere su existencia en el sistema, se utiliza la función `vQueueDelete()`.

* **Sintaxis básica:**
    ```c
    vQueueDelete(xQueue);
    ```
* **Efecto:** Libera la memoria RAM previamente asignada. **Nota crítica:** No se debe intentar acceder a una cola ni realizar operaciones sobre ella (escribir/leer) después de haber sido eliminada, ya que provocará una falla de hardware (*HardFault*).

---

## 2. Gestión Interna y Transferencia de Datos

### ¿Cómo gestiona una Cola los datos que contiene?
FreeRTOS gestiona las colas bajo el principio **FIFO (First In, First Out)** por defecto: el primer dato en ingresar es el primero en ser consumido. 

> **Mecanismo de Copia por Valor (Pass-by-Value):** > Al enviar un dato a la cola, el contenido completo de la variable se copia físicamente bit a bit dentro del espacio de memoria de la cola. Esto significa que la variable original puede ser modificada o salir de su ámbito (*scope*) inmediatamente después de enviarse sin alterar el dato almacenado en la cola. 
> * *Excepción:* Si el tamaño del objeto es muy grande, por eficiencia se suele enviar un **puntero** al dato (Copia por Referencia), debiendo asegurar que la memoria a la que apunta permanezca válida.

### ¿Cómo enviar datos a una Cola?
FreeRTOS provee funciones específicas según el contexto de ejecución y el orden deseado:

1.  **Hacia el final de la cola (FIFO estándar):** `xQueueSend()` o `xQueueSendToBack()`.
2.  **Hacia el principio de la cola (LIFO / Urgencia):** `xQueueSendToFront()`.
3.  **Desde una rutina de interrupción:** `xQueueSendFromISR()`, `xQueueSendToBackFromISR()`, `xQueueSendToFrontFromISR()`.

* **Sintaxis en Tareas:**
    ```c
    BaseType_t xStatus;
    xStatus = xQueueSend(xQueue, &dataToSend, xTicksToWait);
    ```

### ¿Cómo recibir datos de una Cola?
Existen dos formas principales de extraer o leer información:

1.  **Lectura con Extracción (`xQueueReceive`):** Copia el elemento de la cola en el búfer de destino y lo **elimina** de la cola, liberando espacio.
2.  **Lectura de Inspección (`xQueuePeek`):** Copia el elemento del frente de la cola pero **no lo elimina**. Permite que otra tarea (o la misma) vuelva a leer el mismo elemento posteriormente.
3.  **Desde interrupciones:** Se utiliza `xQueueReceiveFromISR()`.

* **Sintaxis en Tareas:**
    ```c
    BaseType_t xStatus;
    xStatus = xQueueReceive(xQueue, &receivedData, xTicksToWait);
    ```

---

## 3. Bloqueos y Mecanismos Avanzados

### ¿Qué significa bloquearse en una Cola?
El bloqueo es el mecanismo mediante el cual FreeRTOS implementa la eficiencia energética y temporal del sistema multitarea. Ocurre en dos situaciones:

* **Bloqueo por Lectura:** Una tarea intenta leer de una cola que está **vacía**.
* **Bloqueo por Escritura:** Una tarea intenta escribir en una cola que está **llena**.

En lugar de consumir ciclos de CPU en un bucle de consulta activa (*polling*), la tarea ingresa al estado **Blocked**. El planificador (*Scheduler*) le quita el control de la CPU y le permite ejecutar a otras tareas listas. La tarea se desbloqueará inmediatamente cuando ocurra una de dos condiciones:
1.  La cola reciba un elemento (si estaba vacía) o libere un espacio (si estaba llena).
2.  Transcurra el tiempo máximo de espera definido por el parámetro `xTicksToWait`.

### ¿Cómo bloquearse en varias Colas?
Para que una única tarea pueda suspenderse a la espera de datos provenientes de múltiples colas simultáneamente, FreeRTOS implementa los **Conjuntos de Colas (Queue Sets)**.

* **Procedimiento:**
    1. Se crea el conjunto con `xQueueCreateSet()`.
    2. Se añaden las colas deseadas al conjunto usando `xQueueAddToSet()`.
    3. La tarea se bloquea llamando a `xQueueSelectFromSet()`. Cuando cualquiera de las colas del conjunto reciba un dato, el conjunto se activará y devolverá el manejador de la cola que provocó el desbloqueo, permitiendo a la tarea saber exactamente de dónde leer.

### ¿Cómo sobrescribir datos en una Cola?
Cuando una cola posee un tamaño estricto de **1 solo elemento** (`uxQueueLength = 1`), se puede utilizar la función `xQueueOverwrite()`.

* **Funcionamiento:** Si la cola está vacía, escribe el dato normalmente. Si la cola ya contiene un dato, reemplaza/pisa el valor viejo con el nuevo. Esta función nunca se bloquea (no tiene parámetro de tiempo de espera) y es ideal para actualizar variables de estado continuas (ej. lecturas de sensores de temperatura).

### ¿Cómo vaciar una Cola?
Para resetear por completo el estado de una cola y remover todos los elementos pendientes de lectura de forma instantánea, se utiliza `xQueueReset()`.

* **Sintaxis:**
    ```c
    xQueueReset(xQueue);
    ```
* **Efecto:** La cola vuelve a quedar vacía (`uxMessagesWaiting = 0`). Cualquier tarea que estuviese bloqueada esperando escribir en ella se desbloqueará si la cola pasa a estar disponible.

---

## 4. Impacto de las Prioridades en el Acceso a Colas

El comportamiento del *Scheduler* ante colas saturadas o vacías depende directamente de la prioridad asignada a las tareas que interactúan con ellas:

| Escenario | Condición del Sistema | Efecto de las Prioridades |
| :--- | :--- | :--- |
| **Múltiples Tareas Bloqueadas por Lectura** | Una cola vacía recibe finalmente un dato. | El kernel desbloquea de manera inmediata a **la tarea de mayor prioridad** de la lista de espera. Si las tareas poseen la misma prioridad, se desbloquea la que lleve más tiempo esperando (orden FIFO de bloqueo). |
| **Múltiples Tareas Bloqueadas por Escritura** | Una cola llena libera un espacio (se lee un dato). | El kernel permite escribir de inmediato a **la tarea de mayor prioridad** que estaba esperando para transmitir. |
| **Tarea Lectora de Alta Prioridad** | Lee continuamente de la cola. | Tan pronto como una tarea de menor prioridad introduce un elemento en la cola, el planificador desaloja (*preemption*) a la tarea de baja prioridad para otorgarle la CPU a la lectora de alta prioridad, procesando el mensaje en tiempo real. |
