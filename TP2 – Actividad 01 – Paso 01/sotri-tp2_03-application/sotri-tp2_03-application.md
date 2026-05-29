# CESE - Sistemas Operativos de Tiempo Real
## Trabajo Práctico N°: 2 - Comunicación de Tareas de FreeRTOS
### Cohorte 26 - Grupo 08
### Responsable de la entrega:
| N° SIU | Apellidos, Nombres    | Fecha | Deadline  |
| :----- | :--------------------- | :------: | :-------: |
| e2620  | Valdez, Boris Cristian | 28/05 | Semana 06 |


# Informe de Actividad 03: Sincronización con Semáforos

## 1. Creación y Uso de Semáforos

### Creación
Para crear semáforos en FreeRTOS, utilizamos las funciones de la API proporcionadas por `semphr.h`.

* **Semáforo Binario:** Se crea utilizando `xSemaphoreCreateBinary()`. 
    * *Nota de depuración:* Por defecto, al crearlo, el semáforo comienza vacío (estado "no disponible").
* **Semáforo Contador:** Se crea utilizando `xSemaphoreCreateCounting(uxMaxCount, uxInitialCount)`.
    * `uxMaxCount`: Define el valor máximo que puede alcanzar el contador.
    * `uxInitialCount`: Define el valor con el que comienza el semáforo.

### Uso (Give y Take)
Ambos tipos de semáforos comparten la misma lógica de gestión:

1.  **`xSemaphoreGive(xSemaphore)`**: Esta función "entrega" o incrementa el semáforo. Si una tarea estaba bloqueada esperando el semáforo, el planificador la despierta.
2.  **`xSemaphoreTake(xSemaphore, xTicksToWait)`**: Esta función intenta "tomar" o decrementar el semáforo. 
    * Si el semáforo está disponible (valor > 0), la tarea lo toma y continúa.
    * Si el semáforo está vacío (valor = 0), la tarea entra en estado **Bloqueado** (Blocked) durante `xTicksToWait` hasta que el semáforo esté disponible. Si usamos `portMAX_DELAY`, la tarea esperará indefinidamente.

---

## 2. Diferencias entre Semáforos Binarios y Contadores

A través de la experimentación y depuración, se observan las siguientes diferencias fundamentales:

| Característica | Semáforo Binario | Semáforo Contador |
| :--- | :--- | :--- |
| **Valor** | Solo 0 (vacío) o 1 (lleno). | Puede tener valores de 0 hasta `N`. |
| **Uso Principal** | **Sincronización:** Avisar a una tarea que un evento ocurrió. | **Gestión de Recursos:** Controlar acceso a un pool de recursos (ej. buffers). |
| **Comportamiento** | Es una señal "Sí/No". | Es una señal de "Cuántos disponibles". |
| **Bloqueo** | La tarea espera a que el evento ocurra (1 señal). | La tarea espera hasta que al menos un recurso esté libre. |

---

## Paso 03: Observaciones del Comportamiento (Sincronización con Semáforo Binario)

Al modificar el mecanismo de comunicación entre `task_btn` y `task_led` para utilizar un Semáforo Binario en lugar de una cola, se observó lo siguiente durante la depuración:

1. **Bloqueo Puro y Eficiencia (Non-Polling):** La tarea `task_led` ahora utiliza `xSemaphoreTake(h_btn_led_bin_sem, portMAX_DELAY)`. Esto significa que la tarea entra en un estado de bloqueo absoluto (*Blocked state*) y no consume ciclos de CPU mientras el botón no sea presionado, optimizando enormemente el sistema frente al uso de banderas (flags) por software.
2. **Sincronización Directa (Trigger):** En el momento exacto en que `task_btn` valida la pulsación (pasando su delay de anti-rebote) y ejecuta `xSemaphoreGive()`, el planificador (Scheduler) de FreeRTOS despierta de inmediato a la tarea del LED para que conmute (*toggle*) su estado. La latencia de sincronización es mínima.
3. **Ausencia de Memoria Acumulativa (Sin Buffering):** La mayor diferencia observada empíricamente respecto a la Cola (Actividad 02) es que el semáforo binario solo tiene valor `0` o `1`. Si la tarea del botón ejecutara múltiples `Give` de manera extremadamente rápida sin que el LED hiciera un `Take`, el semáforo no "cuenta" ni "encola" las pulsaciones (se clava en 1). Esto demuestra que es ideal para sincronización unidireccional de eventos simples, pero no para transferencia de datos en ráfaga.
