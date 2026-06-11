Podés escribir el archivo `soe-tp2_02-application.md` con respuestas técnicas,
# ¿Cómo eliminar una Cola?

Una Cola se elimina utilizando la función:

```c
vQueueDelete( QueueHandle_t xQueue );
````

La función libera la memoria dinámica utilizada por la Cola y deja inválido su handle.

---

# ¿Cómo crear una Cola?

Una Cola se crea utilizando la función:

```c
xQueueCreate()
```

Ejemplo:

```c
QueueHandle_t xQueue;

xQueue = xQueueCreate(
            5,
            sizeof(uint8_t)
         );
```

donde:

* 5: cantidad máxima de elementos almacenados.
* sizeof(uint8_t): tamaño de cada elemento.

La función devuelve un QueueHandle_t que identifica la Cola creada.

---

# ¿Cómo gestiona una Cola los datos que contiene?

Las Colas de FreeRTOS almacenan los datos utilizando una estructura FIFO (First In First Out).

Cuando una Tarea envía datos a la Cola, FreeRTOS copia el contenido en memoria interna de la Cola.

Cuando otra Tarea recibe datos, FreeRTOS copia el elemento desde la Cola hacia la variable destino.

La Cola administra automáticamente:

* memoria interna,
* índices de lectura y escritura,
* sincronización entre Tareas,
* bloqueo y desbloqueo de Tareas.

---

# ¿Cómo enviar datos a una Cola?

Los datos se envían utilizando funciones como:

```c
xQueueSend()
xQueueSendToBack()
xQueueSendToFront()
```

Ejemplo:

```c
uint8_t data = 1;

xQueueSend(
    xQueue,
    &data,
    portMAX_DELAY
);
```

La Tarea puede bloquearse si la Cola se encuentra llena.

---

# ¿Cómo recibir datos de una Cola?

Los datos se reciben utilizando:

```c
xQueueReceive()
```

Ejemplo:

```c
uint8_t rxData;

xQueueReceive(
    xQueue,
    &rxData,
    portMAX_DELAY
);
```

La Tarea puede bloquearse esperando que existan datos disponibles en la Cola.

---

# ¿Qué significa bloquearse en una Cola?

Significa que una Tarea entra en estado Blocked mientras espera:

* espacio libre para escribir en la Cola,
  o
* datos disponibles para leer.

Durante ese tiempo la CPU puede ejecutar otras Tareas.

---

# ¿Cómo bloquearse en varias Cola?

FreeRTOS permite bloquearse sobre múltiples objetos de comunicación utilizando:

```c
Queue Sets
```

mediante las funciones:

```c
xQueueCreateSet()
xQueueAddToSet()
xQueueSelectFromSet()
```

Esto permite que una misma Tarea espere eventos provenientes de múltiples Colas o Semáforos.

---

# ¿Cómo sobrescribir datos en una Cola?

Se utiliza:

```c
xQueueOverwrite()
```

Esta función sobrescribe el dato anterior almacenado en una Cola de longitud 1.

Se utiliza principalmente para compartir el valor más reciente de una variable o evento.

---

# ¿Cómo vaciar una Cola?

Una Cola puede vaciarse utilizando:

```c
xQueueReset()
```

La función elimina todos los elementos almacenados y reinicia el estado interno de la Cola.

---

# ¿Cuál es el efecto de las prioridades de las Tareas al escribir y leer en una Cola?

Cuando varias Tareas esperan acceder a una Cola:

* la Tarea de mayor prioridad tiene preferencia para ejecutarse,
* FreeRTOS desbloquea primero a la Tarea de mayor prioridad,
* una Tarea de baja prioridad puede quedar esperando si existe una Tarea de mayor prioridad lista para ejecutarse.

Esto afecta el orden de ejecución y el tiempo de respuesta del sistema.


---

# Implementacion 

Actualmente, la comunicación entre `task_btn` y `task_led` se realiza a través de una **variable global compartida** (`task_led_dta`), modificada mediante la función `put_event_task_led` en `task_led_interface.c`.

* **El Problema:** Aunque esto funciona en un entorno simple, **no es una práctica segura en sistemas RTOS**. Al acceder a `task_led_dta.flag` y `task_led_dta.event` sin mecanismos de sincronización (como colas o mutexes), te expones a condiciones de carrera (*race conditions*) si el planificador cambia de tarea justo en medio de la escritura/lectura de esos datos.

### El objetivo: Comunicación mediante Colas (Queues)

Para resolver el **Paso 3** de la guía, debes reemplazar esa comunicación insegura por una **Cola de FreeRTOS (`QueueHandle_t`)**. La cola permitirá que `task_btn` envíe eventos de forma segura y que `task_led` los reciba bloqueándose si no hay datos, lo cual es mucho más eficiente para el CPU.

---

### Pasos para implementar la solución

#### 1. Crear la Cola (en `app.c`)

Debes definir el *handle* de la cola y crearla en `app_init`.

```c
/* En app.c - Declaración global */
QueueHandle_t h_queue_led;

/* En app_init() */
h_queue_led = xQueueCreate(10, sizeof(task_led_ev_t));
configASSERT(h_queue_led != NULL); // Validar creación

```

#### 2. Modificar la Interfaz (`task_led_interface.c`)

Ahora, `put_event_task_led` no debe tocar la variable global directamente, sino enviar el evento a la cola.

```c
void put_event_task_led(task_led_ev_t event)
{
    // Enviamos el evento a la cola de forma segura
    xQueueSend(h_queue_led, &event, 0); 
}

```

#### 3. Actualizar `task_led.c` para consumir la cola

La `task_led` ya no debe depender de un `flag` global. En su `task_led_statechart`, debe intentar recibir eventos de la cola.

```c
void task_led_statechart(void)
{
    task_led_ev_t event_recibido;

    // Intentamos recibir un evento de la cola sin bloquear demasiado tiempo
    if (xQueueReceive(h_queue_led, &event_recibido, 0) == pdPASS)
    {
        task_led_dta.event = event_recibido;
        task_led_dta.flag = true;
    }
    
    // ... luego mantienes tu switch case existente, 
    // pero ahora usas el flag actualizado por la cola ...
}

