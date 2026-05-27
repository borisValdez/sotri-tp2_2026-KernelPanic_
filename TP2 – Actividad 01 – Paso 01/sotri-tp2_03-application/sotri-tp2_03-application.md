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