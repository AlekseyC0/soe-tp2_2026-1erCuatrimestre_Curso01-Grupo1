
## 1. Implementación en `soe-tp2_03-application.md`

### Observaciones del Paso 03

* **Mecanismo de comunicación**: Se reemplazó la cola de mensajes por un **Semáforo Binario**.
* **Comportamiento observado**: Al presionar el pulsador, la tarea `task_btn` ejecuta `xSemaphoreGive()`. La tarea `task_led`, que se encontraba bloqueada en `xSemaphoreTake()`, se desbloquea inmediatamente para ejecutar el cambio de estado (`ST_LED_BLINK`).
* **Eficiencia**: Se eliminó el *polling* de estado, logrando que `task_led` pase a estado "Blocked" mientras no hay eventos, reduciendo el consumo de ciclos de CPU de forma drástica comparado con la implementación anterior.
* **Determinismo**: El uso de semáforos binarios asegura una sincronización punto a punto (de tarea a tarea) más directa y con menor *overhead* de memoria que las colas de datos.

---

## 2. Guía Teórica: Semáforos en FreeRTOS

### ¿Cómo crear y usar semáforos?

#### Semáforos Binarios

Son ideales para sincronización de tareas o gestión de exclusión mutua simple.

1. **Creación**:
```c
SemaphoreHandle_t xSemaphore = xSemaphoreCreateBinary();

```


2. **Uso (Give/Take)**:
* **Give**: Libera el semáforo (lo pone en estado "disponible").
`xSemaphoreGive(xSemaphore);`
* **Take**: Intenta tomarlo. Si está en 0, bloquea la tarea.
`xSemaphoreTake(xSemaphore, portMAX_DELAY);`



#### Semáforos Contadores

Se utilizan para manejar recursos limitados donde múltiples instancias del mismo recurso están disponibles (ej. un buffer con 5 espacios libres).

1. **Creación**:
```c
// Parámetros: (máximo valor, valor inicial)
SemaphoreHandle_t xCountingSemaphore = xSemaphoreCreateCounting(5, 0);

```


2. **Uso**:
* Funciona igual que el binario con `xSemaphoreGive` y `xSemaphoreTake`, pero el contador interno se incrementa o decrementa según la cantidad de recursos "dados" o "tomados".



---

### Diferencias entre Semáforos Binarios y Contadores

| Característica | Semáforo Binario | Semáforo Contador |
| --- | --- | --- |
| **Valor máximo** | Solo puede ser 0 o 1. | Puede ser un entero configurado al crearse. |
| **Uso principal** | Sincronización (señalización) y exclusión mutua. | Gestión de recursos múltiples o conteo de eventos. |
| **Estado inicial** | Generalmente vacío (0). | Puede iniciarse en cualquier valor entre 0 y el máximo. |
| **Comportamiento** | Actúa como un interruptor. | Actúa como una reserva de "tokens". |

### Conclusión para tu Bitácora

La migración a semáforos binarios optimiza el uso de recursos del microcontrolador al delegar la gestión de espera al *Scheduler* de FreeRTOS, garantizando que ninguna tarea "gaste" CPU innecesariamente mientras espera por un evento de entrada.
