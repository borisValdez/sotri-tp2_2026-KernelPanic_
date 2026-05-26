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


