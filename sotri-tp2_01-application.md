# CESE - Sistemas Operativos de Tiempo Real
## Trabajo Práctico N°: 2 - Comunicación de Tareas de FreeRTOS
### Cohorte 26 - Grupo 08
### Responsable de la entrega:
| N° SIU | Apellidos, Nombres    | Fecha | Deadline  |
| :----- | :--------------------- | :------: | :-------: |
| e2620  | Valdez, Boris Cristian | 28/05 | Semana 06 |




1. Análisis y explicación del funcionamiento de los archivos adjuntos
startup_stm32f446retx.s (Archivo de arranque / Startup): Este archivo está escrito en lenguaje ensamblador y contiene el código que se ejecuta inmediatamente después de encender o reiniciar el microcontrolador. Su función principal es establecer el entorno de ejecución en C: inicializa el puntero de pila (Stack Pointer), copia las variables inicializadas (sección .data) desde la memoria Flash a la memoria RAM, llena con ceros las variables no inicializadas (sección .bss), y finalmente invoca la función SystemInit antes de saltar a la función main(). Además, define la tabla de vectores de interrupción del sistema.

main.c (Programa principal): Es el punto de entrada de tu aplicación en C. Aquí se inicializan todos los periféricos del hardware (HAL, Relojes del sistema, GPIO y puertos serie UART). Posterior a la inicialización del hardware, se definen y crean los recursos del sistema operativo (en este caso, un hilo llamado defaultTask) usando las envolturas CMSIS-RTOS, y se lanza el planificador llamando a osKernelStart().

stm32f4xx_it.c (Rutinas de servicio de interrupción): Este archivo contiene los "Handlers" o manejadores de las interrupciones del procesador Cortex-M4 y de los periféricos específicos del STM32. Por ejemplo, aquí se encuentra HardFault_Handler para errores críticos y TIM1_UP_TIM10_IRQHandler, la cual deriva la ejecución hacia el manejador de interrupciones de temporizadores de la librería HAL.

FreeRTOSConfig.h (Configuración del RTOS): Es el archivo de cabecera fundamental donde se ajustan los parámetros de FreeRTOS para adaptarlos a la aplicación. Aquí se definen aspectos vitales como la frecuencia del reloj (configCPU_CLOCK_HZ), la velocidad del tick del sistema operativo (configTICK_RATE_HZ a 1000 Hz o 1 ms), el tamaño de la memoria dinámica/estática (Heap) y las prioridades máximas. También se mapean los nombres estándar de las interrupciones de FreeRTOS a las de la arquitectura ARM (ej. #define xPortSysTickHandler SysTick_Handler).

freertos.c (Código de integración de FreeRTOS): En este código generado se implementan funciones adicionales o de soporte para la correcta operación del sistema operativo. En el código proporcionado, incluye la función vApplicationGetIdleTaskMemory(), la cual suministra las estructuras de memoria y el tamaño de pila estática requeridos para que FreeRTOS pueda crear la Tarea Inactiva (Idle Task) por defecto.

2. Evolución de las variables SysTick y SystemCoreClock
Desde el reinicio hasta el bucle principal, estas configuraciones evolucionan así:

SystemCoreClock: Cuando el sistema pasa por Reset_Handler, ejecuta SystemInit (rutina que por lo general configura el reloj interno por defecto). Al entrar al main(), se ejecuta SystemClock_Config(). En esta función se activa y configura el PLL para que el microcontrolador trabaje a su máxima frecuencia deseada utilizando el oscilador interno (HSI). Internamente (y aunque no se muestre explícitamente en el código adjunto, es parte de la librería estándar subyacente de CMSIS), esto actualiza la variable global SystemCoreClock para reflejar la nueva frecuencia. En FreeRTOSConfig.h, esta variable le informa al sistema operativo qué tan rápido está corriendo el microcontrolador (configCPU_CLOCK_HZ = SystemCoreClock).

SysTick: El SysTick del núcleo ARM inicialmente se mantiene deshabilitado. No es sino hasta que se llama a osKernelStart() en el main.c que FreeRTOS toma posesión de este temporizador y lo configura basándose en el SystemCoreClock para interrumpir cada 1 milisegundo, comenzando así a llevar el tiempo del RTOS.

3. Comportamiento del programa desde el inicio (Reset_Handler) hasta el loop principal de la aplicación
El flujo secuencial de la aplicación es el siguiente:

Arranque físico: Al iniciar, se llama a Reset_Handler que establece la ubicación de la pila (_estack) en memoria.

Preparación de memoria: El código en ensamblador copia los valores iniciales de las variables globales desde la memoria Flash hacia la RAM y limpia los espacios restantes de memoria (variables en cero).

Salto a C: Se efectúa un salto (bl main) ingresando a la función principal en main.c.

Inicialización de Hardware: El main() ejecuta HAL_Init() y SystemClock_Config() para configurar temporizadores y la frecuencia del núcleo. Luego inicia los periféricos básicos mediante MX_GPIO_Init() y MX_USART2_UART_Init().

Creación de entorno RTOS: El código indica al sistema operativo qué tareas existirán; se crea explícitamente la defaultTask mediante osThreadCreate().

Arranque del Scheduler: Se invoca a osKernelStart(). En este momento exacto, el planificador (Scheduler) de FreeRTOS asume el control total de la CPU.

Loop principal inalcanzable: Debido a que el planificador nunca retorna (a menos que ocurra un error fatal o falta de memoria), el procesador nunca llega al bucle while (1) ubicado al final del main(). A partir de este momento, el código que se ejecuta cíclicamente es el contenido en el for(;;) de la función StartDefaultTask.

4. Cómo y para qué SysTick y el Timer 1 interactúan con FreeRTOS y con la HAL
(Nota: Aunque la guía menciona "Timer 1 (TIM2)", de acuerdo al código fuente provisto, es el TIM1 el que está configurado como base de tiempo alternativo).

Cuando se utiliza un Sistema Operativo en Tiempo Real (RTOS) junto con las librerías de abstracción de hardware (HAL) de STM32Cube, ocurre un conflicto por el control del temporizador principal:

Interacción de SysTick: El temporizador interno del núcleo ARM (SysTick) es exclusivo para FreeRTOS. Esto se confirma en FreeRTOSConfig.h donde se redirecciona la interrupción con la línea #define xPortSysTickHandler SysTick_Handler. El SysTick le permite al planificador llevar la cuenta de los "ticks", realizar retrasos como osDelay(1000) en las tareas y alternar la ejecución entre hilos del mismo nivel de prioridad (Time Slicing).

Interacción del Timer (TIM1) con la HAL: Como el RTOS monopoliza el SysTick, la librería HAL necesita otra forma de medir el tiempo para sus propios procesos (como los "timeouts" al enviar datos por el puerto serie UART o retardos usando HAL_Delay()). Para solucionar esto, el código asigna el hardware del Timer 1 (TIM1) a la HAL. Cada vez que TIM1 se desborda, detona la interrupción TIM1_UP_TIM10_IRQHandler. Esta llama a la librería, la cual a su vez dispara la función de retrollamada HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim) en el main.c. Si la interrupción proviene de TIM1, se incrementa el contador de la HAL ejecutando la función HAL_IncTick().
