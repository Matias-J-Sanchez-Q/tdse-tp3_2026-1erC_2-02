¡Hola! Claro que sí, es un proyecto excelente y muy completo. Integrar el control de un display LCD, portar código y estructurar la lógica mediante máquinas de estado (statecharts) cubre los pilares fundamentales del desarrollo de sistemas embebidos.

Para organizar tu Trabajo Práctico (TP) de manera profesional y lógica, te sugiero dividirlo en las siguientes tres etapas clave:

## 1. System Setup y Porting de Código C (Hardware Abstraction Layer)

Cuando hablamos de "portar" código (generalmente un driver para el clásico controlador LCD HD44780), el objetivo es que tu código sea independiente del microcontrolador que estés usando. Para lograr esto, debes implementar una **Capa de Abstracción de Hardware (HAL)**.

- **Separación de capas:** Tu código debe tener un archivo de alto nivel (ej. `lcd.c` y `lcd.h`) que contenga la lógica del display, y un archivo de bajo nivel (ej. `lcd_port.c`) que dependa del hardware.
    
- **Funciones a portar:** Para migrar el código a cualquier microcontrolador, solo deberías necesitar reescribir funciones básicas como:
    
    - `LCD_SetDataPins(uint8_t data)`: Configura los pines de datos (D4-D7 o D0-D7).
        
    - `LCD_SetControlPins(bool rs, bool rw, bool en)`: Controla los pines de estado.
        
    - `Delay_ms(uint32_t ms)`: Función de retardo específica del sistema.
        
- **System Setup:** En la inicialización principal de tu sistema (`main.c`), debes configurar los pines del microcontrolador (GPIOs) como salidas digitales antes de llamar a la función de inicialización del LCD.
    

---

## 2. Modelado con Statecharts (Máquinas de Estado)

En sistemas embebidos modernos, quedarse bloqueado esperando en un `Delay_ms()` es una mala práctica. Aquí es donde entra el modelado con **Statecharts** (o Máquinas de Estados Finitos - FSM).

Debes diseñar un diagrama de estados que maneje la inicialización y la escritura del LCD de forma asíncrona. Un modelo básico podría tener los siguientes estados:

- **`STATE_INIT`**: El sistema arranca y comienza la secuencia de inicialización del LCD (que requiere varios comandos con tiempos de espera específicos).
    
- **`STATE_IDLE`**: El LCD está listo y esperando que se envíen datos o comandos desde la aplicación principal.
    
- **`STATE_WRITE_IN_PROGRESS`**: Se está enviando un carácter o comando. Este estado maneja los pulsos del pin `Enable` (EN) sin bloquear el CPU.
    
- **`STATE_ERROR`**: En caso de que falle la comunicación (más aplicable si usas I2C en lugar de paralelo).
    

---

## 3. C Coding (Implementación de la Máquina de Estados)

Una vez que tienes el modelo, la traducción a código C estándar se realiza típicamente usando una estructura de `switch-case` dentro de una función que se llama periódicamente (por ejemplo, dentro de un timer o en el bucle infinito `while(1)`).

Aquí tienes una estructura conceptual de cómo se vería:

C

```
typedef enum {
    LCD_STATE_INIT,
    LCD_STATE_IDLE,
    LCD_STATE_WRITE_IN_PROGRESS
} LCD_State_t;

LCD_State_t currentState = LCD_STATE_INIT;

void LCD_StateMachine_Update(void) {
    switch (currentState) {
        case LCD_STATE_INIT:
            // Lógica de inicialización por pasos...
            if (Init_Complete()) {
                currentState = LCD_STATE_IDLE;
            }
            break;
            
        case LCD_STATE_IDLE:
            // Revisa si hay nuevos datos en el buffer para imprimir
            if (Has_Data_To_Write()) {
                currentState = LCD_STATE_WRITE_IN_PROGRESS;
            }
            break;
            
        case LCD_STATE_WRITE_IN_PROGRESS:
            // Genera el pulso EN y envía los datos
            // Vuelve a IDLE cuando termina
            currentState = LCD_STATE_IDLE;
            break;
    }
}
```

---

Para poder ayudarte de forma más precisa con el código y el diagrama de tu trabajo práctico: **¿Qué microcontrolador o familia de placas (ej. STM32, PIC, Arduino/AVR) estás utilizando para este proyecto?**




El conjunto de archivos proporcionados conforma el código fuente de un sistema embebido "Bare Metal" (sin sistema operativo) que utiliza un enfoque de planificación disparada por eventos y tiempo (Event-Triggered Systems). El diseño se basa en máquinas de estados finitos (FSM) que se ejecutan de manera concurrente y no bloqueante.

A continuación, se detalla el funcionamiento de la arquitectura y sus archivos, seguido de la explicación de las dos funciones solicitadas.

### 1. Análisis de los Archivos del Sistema

- **`app.c` y `app_it.c`**: Constituyen el núcleo de la aplicación. `app.c` contiene el bucle principal o el despachador de tareas (dispatcher). Define una lista de tareas (`task_cfg_list`) y se encarga de ejecutar periódicamente las funciones `_update` de cada tarea registrada, midiendo además tiempos de ejecución (como el "Mejor Caso" BCET y "Peor Caso" WCET). `app_it.c` maneja las interrupciones del sistema, principalmente la interrupción del temporizador del sistema (SysTick) para mantener la cuenta del tiempo global (`g_app_tick_cnt`).
    
- **`systick.c`**: Proporciona funciones de retardo bloqueantes en microsegundos (`systick_delay_us`) interactuando directamente con los registros del temporizador SysTick del hardware.
    
- **`display.h` y `display.c`**: Componen el controlador (driver) de bajo nivel para una pantalla LCD alfanumérica (tipo HD44780 o similar). Permiten inicializar la pantalla en modo de 4 u 8 bits, enviar comandos de control y escribir caracteres usando pines GPIO.
    
- **`task_display_attribute.h` y `task_display_interface.c`**: Definen la estructura de datos, los estados y eventos de la tarea encargada de controlar el display. La interfaz proporciona funciones públicas, como `put_event_task_display()`, que permiten a otras tareas (como `task_test`) enviar de forma segura mensajes o eventos para que se muestren en la pantalla sin colisionar.
    
- **`task_test_attribute.h` y `task_test.c`**: Implementan una tarea de prueba. Aunque `task_test_attribute.h` no se incluye en el texto visible, se deduce que contiene la estructura `task_test_dta_t` que almacena el conteo de ticks y contadores de la tarea. La tarea sirve para inyectar datos en la pantalla de forma periódica.
    

---

### 2. Comportamiento de las Máquinas de Estados

#### Función `void task_test_statechart(void)`

Esta función implementa la lógica de la tarea de prueba mediante un temporizador no bloqueante.

- **Contador general**: En cada ciclo de ejecución, incrementa un contador global interno (`p_task_test_dta->counter++`).
    
- **Temporización**: Utiliza una variable `tick` para generar un retardo no bloqueante. Si el valor actual de `tick` es mayor al mínimo (`DEL_TEST_XX_MIN`), simplemente lo decrementa.
    
- **Disparo de evento**: Cuando el `tick` llega al mínimo, significa que el tiempo de espera ha transcurrido. En ese momento:
    
    1. Restablece el temporizador cargándolo con el valor máximo (`DEL_TEST_XX_MAX`).
        
    2. Envía texto al display utilizando la función de interfaz `put_event_task_display()`, escribiendo "Test Nro: ******" en la primera columna, segunda fila.
        
    3. Calcula un valor basado en el contador, lo convierte a cadena de texto con `snprintf` y lo envía a la pantalla en la posición de la columna 10.
        

#### Función `void task_display_statechart(void)`

Esta función es una Máquina de Estados Finitos (FSM) que se encarga de actualizar físicamente el LCD basándose en los eventos que recibe. Tiene los siguientes estados principales:

- **`ST_DSP_IDLE` (Reposo)**: La máquina está a la espera de un evento. Constantemente evalúa si hay una bandera activa (`p_task_display_dta->flag == true`) y si el evento indicado es una petición de actualización (`EV_DSP_UPDATE`). De ser así, cambia su estado a `ST_DSP_UPDATE`.
    
- **`ST_DSP_UPDATE` (Actualización)**: Cuando ingresa a este estado, procesa la actualización de la pantalla.
    
    1. Apaga la bandera de evento (`p_task_display_dta->flag = false`) para indicar que el evento está siendo atendido.
        
    2. Utiliza las funciones del driver (`displayCharPositionWrite` y `displayStringWrite`) para volcar el contenido de su memoria de video virtual interna (`ddram`) al hardware real. Escribe primero la fila 0 y luego la fila 1.
        
    3. Finalmente, retorna su estado a `ST_DSP_IDLE` para quedar a la espera del próximo evento de actualización.
        
- **`default`**: Un mecanismo de seguridad por si la FSM entra en un estado corrupto o no válido, devolviéndola inmediatamente al estado `ST_DSP_IDLE` y limpiando sus banderas.

# Análisis de Tiempos - Actividad 01

**Unidad de medida:** Microsegundos (us)

**Valores obtenidos de task_dta_list tras varias ejecuciones:**
* task_test (Index 0) WCET:  35 us
* task_display (Index 1) WCET: 6267 us
