# CESE - Sistemas Operativos de Tiempo Real
## Trabajo Práctico N°: 2 - Comunicación de Tareas de FreeRTOS
### Cohorte 26 - Grupo 08
### Responsable de la entrega:
| N° SIU | Apellidos, Nombres    | Fecha | Deadline  |
| :----- | :--------------------- | :------: | :-------: |
| e2620  | Valdez, Boris Cristian | 28/05 | Semana 06 |


### Actividad 01 - Análisis del Código Fuente Base (`sotri-tp2_01-application`)

---

## 1. Análisis y Explicación del Funcionamiento de los Archivos Fuente

### A. `startup_stm32f446retx.s`
Es el archivo de arranque del sistema escrito en lenguaje ensamblador (`ARM assembly`). Sus funciones principales son:
* **Definición de la Tabla de Vectores:** Define la tabla `g_pfnVectors` que asocia cada interrupción física o excepción del núcleo (como Reset, HardFault, SysTick) con su respectiva función de servicio (ISR).
* **Configuración del Entorno de Ejecución en C:** Inicializa el puntero de pila (`Stack Pointer` o `SP`) apuntando al final de la RAM (`_estack`).
* **Inicialización de Memoria:** Copia los valores iniciales de las variables globales de la sección `.data` desde la memoria Flash a la RAM, y limpia (escribe con ceros) la sección `.bss` correspondiente a variables no inicializadas.
* **Salto al Código de Aplicación:** Llama a la función de inicialización del sistema de bajo nivel `SystemInit` y posteriormente salta al punto de entrada del programa en C (`main`).

### B. `main.c`
Es el archivo principal que gobierna el ciclo de vida inicial de la aplicación. Se encarga de:
* **Inicialización de la HAL:** Llama a `HAL_Init()` para configurar los periféricos básicos de abstracción de hardware, la estructura de prioridades de las interrupciones (NVIC) y la base de tiempo.
* **Configuración del Árbol de Relojes:** Ejecuta `SystemClock_Config()` para establecer las frecuencias de operación del núcleo y de los buses periféricos (AHB/APB).
* **Inicialización de Periféricos Propios:** Llama a las funciones autogeneradas como `MX_GPIO_Init()` y `MX_USART2_UART_Init()` para dejar los pines de E/S y el puerto serie listos para operar.
* **Creación de Objetos del RTOS:** Mediante la API de abstracción CMSIS-RTOS, define y reserva la memoria para el hilo por defecto (`defaultTask`) usando `osThreadCreate()`.
* **Lanzamiento del Sistema Operativo:** Llama a `osKernelStart()` para delegar el control de la CPU al Planificador (Scheduler).

### C. `stm32f4xx_it.c`
Contiene los manejadores de interrupciones (`Interrupt Service Routines - ISR`) del sistema. En este archivo se encuentran tanto las excepciones del núcleo ARM (como `NMI_Handler`, `HardFault_Handler`, etc.) como las interrupciones de periféricos específicos de ST. 
* En este diseño particular, destaca el `TIM1_UP_TIM10_IRQHandler`, que captura los desbordamientos de hardware del temporizador TIM1 y los deriva a la función abstracta `HAL_TIM_IRQHandler()`.

### D. `FreeRTOSConfig.h`
Es el archivo de cabecera que parametriza y adapta el comportamiento del núcleo de FreeRTOS. Define directivas de precompilación críticas como:
* `configCPU_CLOCK_HZ`: Frecuencia del reloj del procesador (enlazada dinámicamente a la variable `SystemCoreClock`).
* `configTICK_RATE_HZ`: Definida en `1000`, lo que implica que el sistema operativo genera un "Tick" (conmutación y revisión de tareas) cada **1 milisegundo**.
* `configMAX_PRIORITIES`: Define el número de niveles de prioridad de tareas disponibles (en este caso, `7`).
* **Mapeo de interrupciones:** Redirecciona los vectores nativos de FreeRTOS hacia los nombres de vectores de interrupción de ARM CMSIS (ej. `#define vPortSVCHandler SVC_Handler` y `#define xPortPendSVHandler PendSV_Handler`).

### E. `freertos.c`
Contiene la lógica de inicialización y soporte de las funciones de FreeRTOS generadas por el entorno gráfico STM32CubeMX. Debido a que el proyecto utiliza **asignación estática de memoria** para ciertas tareas base, este archivo implementa la función regulatoria obligatoria `vApplicationGetIdleTaskMemory()`. Esta función provee de manera estática las estructuras de control (`StaticTask_t`) y los buffers de pila (`StackType_t`) requeridos por el sistema operativo para crear la *Tarea Inactiva* (`Idle Task`).

---

## 2. Evolución de las Variables `SysTick` y `SystemCoreClock`

A lo largo de la secuencia de arranque, el comportamiento y configuración de estas variables evoluciona de la siguiente manera:

| Etapa del Programa | Estado de `SystemCoreClock` | Estado de Hardware `SysTick` |
| :--- | :--- | :--- |
| **Reset_Handler** | Posee un valor por defecto inicial (generalmente ligado al oscilador interno básico HSI de 16 MHz). | Deshabilitado / Sin configurar por hardware. |
| **Post `SystemClock_Config()`** | Se actualiza internamente al valor de la frecuencia máxima configurada mediante el PLL (ej. 180 MHz para STM32F446). | Sigue deshabilitado para el sistema operativo. La HAL utiliza el **TIM1** como base alternativa. |
| **Post `osKernelStart()`** | Se mantiene fijo en el valor configurado. Es leído por FreeRTOS para calcular las temporizaciones. | **Activado e Inicializado**. El planificador configura el registro de recarga de SysTick para interrumpir exactamente cada 1 ms en base a `SystemCoreClock`. |

---

## 3. Comportamiento Secuencial del Programa (Desde Reset hasta el Bucle Principal)

La línea de tiempo lógica de la CPU desde que recibe energía o se presiona el botón Reset se resume en los siguientes pasos consecutivos:

1. El hardware lee el vector de Reset y salta a la dirección de **`Reset_Handler`** en `startup_stm32f446retx.s`.
2. Se configura el puntero de pila y se realiza la inicialización física de la memoria (copia de `.data` y vaciado de `.bss`).
3. Se invoca a `SystemInit` para configuraciones básicas de registros del procesador y se salta inmediatamente a **`main()`**.
4. En `main()`, se inicializa el ecosistema ST con `HAL_Init()` y se eleva la frecuencia de reloj al máximo mediante `SystemClock_Config()`.
5. Se configuran los puertos de entrada/salida y comunicaciones (`MX_GPIO_Init()`, `MX_USART2_UART_Init()`).
6. Se solicita formalmente al kernel la creación del hilo de ejecución por defecto mediante `osThreadCreate(osThread(defaultTask), NULL)`. En este punto, FreeRTOS ubica la tarea en la lista de tareas listas (`Ready List`).
7. El programa ejecuta **`osKernelStart()`**.
8. **Punto de Quiebre / Bloqueo del Flujo Lineal:** Al arrancar el planificador, FreeRTOS configura las interrupciones del sistema, activa el temporizador del núcleo (SysTick) y le otorga el control de la CPU a la tarea de mayor prioridad disponible (en este caso, `defaultTask`).
9. **Consecuencia Práctica:** Como el planificador toma el control absoluto de los saltos de CPU, **el programa nunca llega a ejecutar la línea `while (1)`** ubicada al final de la función `main()`. El flujo se vuelve concurrente dentro de la función cíclica `StartDefaultTask()`.

---

## 4. Interacción de `SysTick`, `TIM1` con FreeRTOS y las Librerías HAL

En un sistema convencional de STM32 sin RTOS, la librería HAL monopoliza el uso del temporizador interno **SysTick** de ARM para generar retardos (`HAL_Delay()`) y gestionar los límites de tiempo (*timeouts*) de periféricos. Al incorporar FreeRTOS, se produce un conflicto, ya que el sistema operativo requiere de forma obligatoria y exclusiva el SysTick para conmutar tareas y llevar el tiempo del sistema.

Para resolver este conflicto, el proyecto implementa la metodología recomendada por STMicroelectronics:

### A. Rol del `SysTick` con FreeRTOS
El temporizador SysTick se asigna de forma exclusiva al núcleo de FreeRTOS. Corre de manera ininterrumpida y genera una excepción de hardware cada **1 milisegundo**. Cada vez que esta excepción ocurre:
* Se ejecuta el manejador del port de FreeRTOS.
* El sistema incrementa el contador de ticks global (`xTickCount`).
* Se evalúa si alguna tarea bloqueada por tiempo debe despertar o si corresponde realizar un cambio de contexto por prioridad (*Time Slicing*).

### B. Rol del `TIM1` con la Librería HAL
Para que la HAL mantenga su independencia y sus funciones internas no sufran interferencias ni retrasos causados por el kernel del sistema operativo, se configura un temporizador de hardware de propósito general—en este caso específico, el **TIM1**—como la nueva base de tiempo de la HAL (*Timebase Source*).

* **Mecanismo de Interacción:** Cada 1 milisegundo, el hardware de TIM1 se desborda y genera una interrupción que es capturada en `stm32f4xx_it.c` dentro de `TIM1_UP_TIM10_IRQHandler()`. Esta rutina invoca las funciones de actualización interna de ST:
  
```c
void TIM1_UP_TIM10_IRQHandler(void)
{
  HAL_TIM_IRQHandler(&htim1);
}

## Actividad 02 - Paso 08: Análisis del Código de la Aplicación (`app/` y `task/`)

---

## 1. Análisis Funcional de los Archivos de la Aplicación

### A. `app.c` (Inicializador de la Aplicación)
Es el módulo encargado de orquestar la creación de los componentes de software de alto nivel una vez que el hardware básico ya fue inicializado. 
* **Función Principal (`app_init`):** Invoca a `app_it_init()` para preparar las interrupciones de la aplicación. Luego, utiliza la API nativa de FreeRTOS (`xTaskCreate`) para instanciar las dos tareas principales del sistema: `task_btn` ("Task BTN") y `task_led` ("Task LED").
* **Configuración de Tareas:** A ambas tareas se les asigna una prioridad de `tskIDLE_PRIORITY + 1ul` (prioridad baja, justo por encima de la tarea inactiva) y se definen sus tamaños de pila (*Stack Depth*) como el doble del mínimo (`2 * configMINIMAL_STACK_SIZE`). El archivo hace uso de `configASSERT(pdPASS == ret)` para detener inmediatamente la CPU si el sistema operativo se queda sin memoria RAM (Heap) al intentar crear estos hilos.

### B. `app_it.c` (Manejador de Interrupciones de la Aplicación)
Este archivo centraliza las funciones de respuesta a eventos externos por interrupción que pertenecen a la lógica de la aplicación.
* **Mecanismo de Captura:** Implementa la función de retrollamada de la HAL de ST: `HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)`. Cuando se presiona un botón físico configurado en modo interrupción externa (EXTI), el hardware salta aquí. 
* **Lógica Interna:** El código evalúa si el pin que generó la interrupción coincide con el identificador del botón (`BTN_Pin`). Aunque en esta etapa base el botón se procesa principalmente por consulta periódica (*polling*), este archivo deja preparada la estructura para la migración hacia interrupciones en la Actividad 04.

### C. `task_btn.c` (Tarea de Gestión del Botón)
Implementa el ciclo de vida y comportamiento del botón de la placa de desarrollo. Su objetivo principal es leer el estado físico del pin del botón, filtrar el ruido eléctrico mediante software (*anti-bounce*) y notificar las pulsaciones válidas.
* **Algoritmo:** Utiliza una estructura de datos interna (`task_btn_dta`) que almacena el estado actual de la máquina de estados, el evento detectado y una marca de tiempo (*tick*) obtenida con `xTaskGetTickCount()`. Cuando detecta una transición estable y válida, registra el log `BTN PRESSED` mediante el logger de la aplicación y despacha el evento de parpadeo invocando a `put_event_task_led(EV_LED_BLINK)`.

### D. `task_led.c` (Tarea de Gestión del LED)
Representa el hilo encargado de controlar el actuador visual (el LED indicador de la placa).
* **Control de Tiempo Preciso:** En su bucle infinito, la tarea utiliza `vTaskDelayUntil(&last_wake_time, LED_TICK_DEL_MAX)`, asegurando que la tarea se ejecute de manera estrictamente periódica cada 50 milisegundos, sin importar el tiempo de procesamiento interno que consuma la CPU.
* **Lógica:** Evalúa cíclicamente los cambios en la estructura de datos `task_led_dta`. Si el indicador `flag` es verdadero y el evento coincide con `EV_LED_BLINK`, cambia su estado interno y comanda al hardware mediante `HAL_GPIO_WritePin` para encender o apagar el pin físico asociado al LED.

### E. `task_led_interface.c` (Interfaz de Comunicación)
Este archivo actúa como una capa de abstracción o "puente" para que otras tareas interactúen con el LED sin necesidad de conocer los detalles de bajo nivel de su máquina de estados.
* **Funcionamiento:** Implementa la función pública `put_event_task_led(task_led_ev_t event)`. Cuando es llamada (por ejemplo, desde `task_btn.c`), escribe directamente el evento recibido en la variable global `task_led_dta.event` y cambia el valor de `task_led_dta.flag = true` para indicarle a la tarea del LED que tiene un comando pendiente por procesar.

### F. `freertos.c` (Hooks y Ganchos del Sistema Operativo)
Contiene las funciones complementarias de diagnóstico y telemetría de FreeRTOS que se ejecutan bajo condiciones específicas del sistema:
* **`vApplicationIdleHook`:** Se ejecuta de manera continua cada vez que no hay ninguna tarea de usuario lista para correr. Incrementa la variable global `g_task_idle_cnt`, la cual sirve para medir el porcentaje de tiempo libre o inactividad de la CPU.
* **`vApplicationTickHook`:** Se ejecuta dentro del contexto de la interrupción del `SysTick` cada 1 ms. Incrementa el contador `g_app_tick_cnt` para métricas temporales de la aplicación.
* **`vApplicationStackOverflowHook`:** Es un mecanismo de seguridad crítico. Si alguna de las tareas se excede del tamaño de pila asignado (`2 * configMINIMAL_STACK_SIZE`), el kernel detecta la corrupción de memoria, salta a este gancho y detiene la ejecución para evitar comportamientos erráticos.

---

## 2. Arquitectura de Comunicación e Interconexión entre Tareas

En este proyecto base (`sotri-tp2_01-application`), el flujo de datos e instrucciones sigue un patrón de **Diseño Orientado a Eventos con Variables Globales Compartidas**.

[ task_btn ] ---> Llama a ---> [ put_event_task_led() ] ---> Modifica ---> [ task_led_dta ] <--- Evalúa <--- [ task_led ]


1. La tarea `task_btn` procesa la entrada del usuario de manera aislada.
2. Al detectar una pulsación, delega la responsabilidad de comunicación a la función interfaz `put_event_task_led()`.
3. Esta función modifica directamente una estructura de memoria global compartida (`task_led_dta`).
4. En el siguiente ciclo de reactivación (cada 50 ms), la tarea `task_led` lee esa misma estructura global, detecta el cambio de bandera (`flag = true`), consume el evento y modifica el estado del periférico.

---

## 3. Análisis de las Máquinas de Estado (Statecharts)

Ambas tareas basan su funcionamiento en el patrón de diseño de **Máquinas de Estado Ejecutadas hasta la Finalización (*Run-to-Completion Statecharts*)**.

### Máquina de Estados del Botón (`task_btn.c`)
Está diseñada para discriminar falsos contactos y ruidos eléctricos mediante transiciones temporizadas:
* **`ST_BTN_UP` (Reposo/Suelto):** Estado inicial. El pin lee un nivel alto constante. Si se detecta un flanco descendente (`EV_BTN_DOWN`), guarda el tiempo actual del sistema operativo y transiciona a `ST_BTN_FALLING`.
* **`ST_BTN_FALLING` (Validación de Bajada):** Espera que transcurra un tiempo de guarda máximo (`DEL_BTN_MAX`). Si al cumplirse el tiempo el botón sigue presionado, se confirma que no es un ruido eléctrico, genera el evento hacia el LED y pasa a `ST_BTN_DOWN`. Si el botón se soltó antes, asume que fue ruido y regresa a `ST_BTN_UP`.
* **`ST_BTN_DOWN` (Presionado Estable):** Se mantiene aquí mientras el usuario sostenga el botón físico. Al detectar la liberación (`EV_BTN_UP`), guarda el tiempo y pasa a `ST_BTN_RISING`.
* **`ST_BTN_RISING` (Validación de Subida):** Monitorea el desprendimiento del botón para evitar falsas pulsaciones dobles causadas por el rebote mecánico del resorte del pulsador. Al estabilizarse, regresa al estado de reposo `ST_BTN_UP`.

### Máquina de Estados del LED (`task_led.c`)
Es una máquina reactiva simple que responde a las solicitudes validadas por el botón:
* **`ST_LED_OFF` (Apagado):** El LED permanece apagado. Si la bandera de la interfaz es verdadera (`flag == true`) y el evento recibido es `EV_LED_BLINK`, la tarea apaga la bandera para consumir el comando, enciende el LED físico usando la HAL, inicializa el temporizador y pasa a `ST_LED_BLINK`.
* **`ST_LED_BLINK` (Parpadeo Cíclico):** El LED conmuta su estado de manera regular. Si se recibe un evento de apagado (`EV_LED_OFF`), apaga el periférico y regresa a `ST_LED_OFF`.

---

## 4. Diagnóstico Crítico: Condición de Carrera (*Race Condition*)

A pesar de que el código base funciona bajo condiciones ideales de laboratorio, presenta un **defecto de diseño crítico** desde la perspectiva de los Sistemas Operativos en Tiempo Real: **La falta de exclusión mutua**.

* **El Problema:** La función `put_event_task_led()` modifica las variables `task_led_dta.event` y `task_led_dta.flag` de manera directa en la memoria RAM. Al mismo tiempo, la tarea `task_led` lee y escribe sobre esos mismos campos de memoria dentro de su bucle.
* **El Peligro (Condición de Carrera):** Si las prioridades de las tareas cambiasen o si esta función se invocara desde una rutina de interrupción (ISR) en medio de la lectura de la tarea del LED, se podría producir una **corrupción de datos**. Por ejemplo, la tarea del LED podría leer la bandera `flag = true` pero evaluar un evento incompleto o viejo si es interrumpida a mitad de la escritura.
* **Conclusión:** Este diseño evidencia de forma didáctica por qué **no se deben utilizar variables globales desprotegidas** para comunicar tareas en un RTOS. Esto justifica y sienta las bases para las siguientes actividades del práctico, donde se reemplazará este mecanismo inseguro por primitivas seguras de FreeRTOS como **Colas de Mensajes (`Queues`)** y **Semáforos Binarios**.

