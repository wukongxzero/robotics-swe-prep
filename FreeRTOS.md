---
type: concept
domain: Real-time & embedded
status: drafted
last-reviewed: 2026-06-02
tags: [freertos, embedded, arduino, rtos, tasks, queues]
---

# FreeRTOS

> [!question] Explain it cold
> - What is FreeRTOS and where does it run?
> - How do you share data safely between two FreeRTOS tasks?
> - How is a FreeRTOS queue different from a mutex?

---

## Core idea

FreeRTOS is a lightweight preemptive RTOS that runs on microcontrollers (AVR, ARM Cortex-M, ESP32, etc.). It gives you tasks, queues, semaphores, and mutexes on hardware that has no OS. Arduino Mega, ESP32, and Teensy all support it. Directly relevant to WALL-E's Arduino stack.

---

## Tasks

```c
#include <FreeRTOS.h>
#include <task.h>

// Task function — must be infinite loop
void motorTask(void *pvParameters) {
    while(1) {
        // read motor commands, drive PWM
        vTaskDelay(pdMS_TO_TICKS(20));  // 50Hz, yields CPU
    }
}

void sensorTask(void *pvParameters) {
    while(1) {
        // read encoders, compute odom
        vTaskDelay(pdMS_TO_TICKS(10));  // 100Hz
    }
}

void setup() {
    // Create tasks
    xTaskCreate(motorTask,  "Motor",  256, NULL, 2, NULL);  // priority 2
    xTaskCreate(sensorTask, "Sensor", 256, NULL, 3, NULL);  // priority 3 (higher)
    vTaskStartScheduler();  // hand control to RTOS — never returns
}

void loop() {}  // never runs — RTOS owns the CPU
```

---

## Queues — safe data passing between tasks

```c
#include <queue.h>

QueueHandle_t motorQueue;

typedef struct {
    int16_t left_pwm;
    int16_t right_pwm;
} MotorCmd;

void setup() {
    motorQueue = xQueueCreate(5, sizeof(MotorCmd));  // 5-item queue
}

// Producer task (e.g. serial receive task)
void serialTask(void *pvParams) {
    MotorCmd cmd;
    while(1) {
        // parse serial data into cmd...
        xQueueSend(motorQueue, &cmd, 0);  // non-blocking send
    }
}

// Consumer task (motor driver)
void motorTask(void *pvParams) {
    MotorCmd cmd;
    while(1) {
        if(xQueueReceive(motorQueue, &cmd, pdMS_TO_TICKS(20))) {
            // apply cmd to PWM
        } else {
            // timeout — no command, stop motors (watchdog)
        }
    }
}
```

**Queue vs shared variable:** queue is thread-safe by design. Shared variable needs a mutex.

---

## Mutex — protecting shared resources

```c
#include <semphr.h>

SemaphoreHandle_t serialMutex;

void setup() {
    serialMutex = xSemaphoreCreateMutex();
}

void taskA(void *pvParams) {
    while(1) {
        if(xSemaphoreTake(serialMutex, pdMS_TO_TICKS(10))) {
            Serial.println("taskA");           // protected access
            xSemaphoreGive(serialMutex);
        }
    }
}
```

---

## Semaphore — signaling between tasks

```c
SemaphoreHandle_t dataReady;

void setup() {
    dataReady = xSemaphoreCreateBinary();
}

// ISR or producer
void ISR() {
    BaseType_t woken = pdFALSE;
    xSemaphoreGiveFromISR(dataReady, &woken);
    portYIELD_FROM_ISR(woken);
}

// Consumer
void processTask(void *pvParams) {
    while(1) {
        xSemaphoreTake(dataReady, portMAX_DELAY);  // block until ISR fires
        // process data
    }
}
```

---

## FreeRTOS on Arduino Mega (WALL-E relevance)

The Mega currently runs a bare-metal loop. FreeRTOS would give:
- Separate tasks for motor control, encoder reading, serial comms
- Queue-based command passing (no shared variable bugs)
- Built-in watchdog via task timeout

```
Task priorities (higher number = higher priority):
  3 — Encoder ISR handler (time-critical)
  2 — Motor command consumer
  1 — Serial comms (ROS ↔ Mega)
  0 — Idle task (FreeRTOS built-in)
```

---

## Memory: stack size

Each task needs its own stack. On Mega (8KB SRAM total):
- Keep stacks small: 128–256 words per task
- `configMINIMAL_STACK_SIZE` = 128 words minimum
- Monitor with `uxTaskGetStackHighWaterMark()`

---

## Key API summary

| Function | What it does |
|----------|-------------|
| `xTaskCreate()` | Create a task |
| `vTaskDelay(ticks)` | Yield for N ticks |
| `xQueueCreate()` | Create a queue |
| `xQueueSend()` | Send to queue (blocks if full) |
| `xQueueReceive()` | Receive from queue (blocks if empty) |
| `xSemaphoreCreateMutex()` | Create mutex (priority inheritance) |
| `xSemaphoreCreateBinary()` | Create binary semaphore (signaling) |
| `xSemaphoreTake()` | Acquire semaphore/mutex |
| `xSemaphoreGive()` | Release semaphore/mutex |
| `vTaskStartScheduler()` | Hand control to RTOS |

---

## Links
- Related: [[RTOS Fundamentals]], [[AVR Register Programming]], [[AVR Peripherals]], [[Serial Packet Protocols]], [[WALL-E V3]]
- Parent: [[00 Knowledge Map]]

---

#flashcards

FreeRTOS queue vs mutex — when do you use each? ? Queue: pass data between tasks safely (producer-consumer pattern). Mutex: protect a shared resource from concurrent access. Queue is the preferred pattern — avoids shared state entirely. Mutex is for when you must share state (e.g. a hardware peripheral).

How do you send data from an ISR to a FreeRTOS task? ? Use `xSemaphoreGiveFromISR()` or `xQueueSendFromISR()` (not the regular versions — ISRs can't block). Call `portYIELD_FROM_ISR(higherPriorityWoken)` at the end to immediately schedule the unblocked task.

What happens if a FreeRTOS task never calls vTaskDelay or blocks? ? It monopolizes the CPU — all lower-priority tasks starve. The idle task never runs, which means memory cleanup and watchdog reset never happen. Always yield in the main loop, even with `vTaskDelay(1)`.

Why does FreeRTOS mutex support priority inheritance but binary semaphore does not? ? Mutex has an owner (the task that took it), so the scheduler knows whose priority to boost when a higher-priority task is waiting. Semaphores have no owner — any task can give them — so there's no concept of "boost the holder."
