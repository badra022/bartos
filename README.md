# BARTOS

Real time operating system implementation for ARM Cortex-M processors

This RTOS is a Preemptive os that can execute the higher priority tasks on time, and switch the context between same priority tasks using round-robin algorithm. this is based on a ready queue that stores all the task control blocks and ne managed each system tick (based on user configurations) to switch to the currently high priority task.

many mechanisms like **semaphores**, **delays** and **message queueing** are implemented as well.

systick and supervisor call interrupts are reserved for rtos usage.

flexible blocking techniques are implemented in semaphores and message queues, so that when some task block on attempt to get a semaphore or send/receive data into the queue
, you can select one of three options:

* block the task without any timeout.
* block the task with specified timeout.
* don't use timout and return immediately without performing the request.

tasks that send data into the message queue are higher priority than the tasks that receive from the queue, unless the queue is full

delay API can move the task from the ready queue to the suspending queue and wake it up again after specific number of ticks. this gives lower priority tasks more chance to run until the high priority tasks wake up from suspension.

This system is designed for 8MHZ system clock, you can configure it to use any other values of frequencies using RCC configurations beside the ```CLOCK_SOURCE``` and ```SYSTEM_TICKS_PER_SEC``` in ```bartos_config.h``` file.

## APIs


### Task management

```c 
unsigned char BARTOS_createTask(void (*task)(void), unsigned char priority)
```

This API creates a task control block (TCB) for the  passed task function.

typically the TCB consists of:
* pointer to task function
* priority of the task
* last saved stack pointer
* state of the task ```[READY, SUSPENDED, BLOCKED, TERMINATED]```
* pointer to the next lower priority TCB ```[priority queue]```
* pointer to the previous higher priority TCB ```[priority queue]```
* message status if the task have/need a message ```[SENDER, RECEIVER]```
* pointer to the block queue of the semaphores that blocked it ```[if it's blocked]```
* pointer to the block queue of the message queue that blocked it ```[if it's blocked]```

The new TCB is created at compile time, fixed number of TCBs is defined in ```bartos_config.h``` file as ```MAX_NUMBER_OF_TASKS```

Stack size is not dynamic, it must be configured the same for all tasks, you can configure it in ```bartos_config.h``` file as ```STD_STACK_SIZE```

priority of zero is prevented for any task.

```MAX_TASK_PRIORITY``` in the config file defines the maximum priority you can attempt to give the created task.

This API returns ```OK``` upon success execution, other codes returned indicate errors


```c 
void BARTOS_endTask(void)
```

This API is considered the exit routine for any task, it set caller task state to ```TERMINATED``` and switch to another task context.

This API can only end the caller task, you cannot end another task using it.

once you end some task, it's like deleting it. it's TCB space is considered free to allocate another newly created TCBs.

upon the call of this API, the scheduler is called to execute the next higher priority task in the ready queue.


```c 
void BARTOS_start(void)
```

This API starts the rtos, it starts with the highest task in the ready queue.

```c
void BARTOS_delayTask(unsigned int ticks)
```

This API suspend the caller task to specific number of system ticks.

The caller task is removed from the ready queue and enqueued into the suspend queue upon the call of this API, then a timer callback function is called after the number of ticks pass to wake the task again and return it the the ready queue.

```c
void BARTOS_IntEnterRoutine(void)
```

This API must be called in the beginning of any ISR that wish to use os Primitive or attempt to change thread's state or create new threads.

```c
void BARTOS_IntExitRoutine(void)
```
This API must be called at the end of any ISR that wish to use os Primitive or attempt to change thread's state or create new threads.


### Message Queues

```c
msgQueueHandler_dtype BARTOS_createQueue(unsigned char* container, unsigned int capacity)
```

This API creates a new message queue using the passed ```container``` and ```capacity```.

container must be declared globally in the file you need to use it in.

you can declare a container of any size you want, but you must select a capacity for the queue implementation that is lower than or equal to container's size

example:
```
#include "bartos.h"

u8 rcv_buffer[100];
msgQueueHandler_dtype rcv_queue;

int main(void){
    rcv_queue = BARTOS_createQueue(rcv_buffer, 50);
    BARTOS_createTask(task1, 4);
    BARTOS_createTask(task2, 5);
    ....
    while(1){
        BARTOS_start();
    }
}

void task1(void){
    ...
}

void task2(void){
    ...
}
```


```c
unsigned char BARTOS_QueueGet(msgQueueHandler_dtype handle, unsigned int timeout, unsigned char* return_ptr)
```

This API attempts to get data item from the message queue, if the queue is empty, the task is blocked using the passed timeout
* block using timeout if ```timeout > 0```
* block without any timeout if ```timeout = 0```
* don't block and return error immediately if ```timeout = -1```

note that in interrupts context, you must call this API using ```timeout = -1```.

if this task is blocked, it's removed from the ready queue and inserted into receiver block queue. this block queue is sorted based on descending priority, thus, it's not guaranteed to wake the first task that blocked on receiving request as it may not be the highest priority blocked on the queue.

upon successful get attempt for this task, it's returned into the ready queue, then it checks for another sender tasks that is blocked in the sender queue and perform it's request if the queue is not full. if the queue is full, it checks for any other receiver tasks blocked in the receiver block queue to perform it.

```c
unsigned char BARTOS_QueuePut(msgQueueHandler_dtype handle, unsigned int timeout, unsigned char data)
```
This API attempts to put data item into the message queue, if the queue is full, the task is blocked using the passed timeout
* block using timeout if ```timeout > 0```
* block without any timeout if ```timeout = 0```
* don't block and return error immediately if ```timeout = -1```

note that in interrupts context, you must call this API using ```timeout = -1```.

if this task is blocked, it's removed from the ready queue and inserted into sender block queue. this block queue is sorted based on descending priority, thus, it's not guaranteed to wake the first task that blocked on sending request as it may not be the highest priority blocked on the queue.

upon successful put attempt for this task, it's returned into the ready queue, then it checks for another sender tasks that is blocked in the sender queue and perform it's request if the queue is not full. if the queue is full, it checks for any other receiver tasks blocked in the receiver block queue to perform it.

### Semaphores

```c
semphrHandler_dtype BARTOS_createBinarySemaphore(void)
```

This API creates semaphore datatype dynamically in the heap, binary semaphore is a special type of counting semaphore where the initial count is 1 and maximum count is 1.



```c
semphrHandler_dtype BARTOS_createCountingSemaphore(unsigned char count, unsigned char maxCount)
```

This API creates semaphore datatype dynamically in the heap,
user can pass whatever initial count and maximum count he want.

counting semaphores can be used in **counting events** and **managing resources**.

```c
unsigned char BARTOS_semaphoreGet(semphrHandler_dtype handle, unsigned int timeout)
```

This API attempts to decrement the count of the semaphore, if the count is already zero, it's blocked using the passed timeout
* block using timeout if ```timeout > 0```
* block without any timeout if ```timeout = 0```
* don't block and return error immediately if ```timeout = -1```

note that in interrupts context, you must call this API using ```timeout = -1```.

This API returns ```OK``` upon success get attempt, otherwise it returns another codes.

block queue is a priority queue where higher priority suspended tasks is served first.

```c
unsigned char BARTOS_semaphorePut(semphrHandler_dtype handle)
```
This API attempts to increment the count of the semaphore, if the count is already maximum, it's blocked using the passed timeout
* block using timeout if ```timeout > 0```
* block without any timeout if ```timeout = 0```
* don't block and return error immediately if ```timeout = -1```

note that in interrupts context, you must call this API using ```timeout = -1```.

This API returns ```OK``` upon success put attempt, otherwise it returns another codes.

block queue is a priority queue where higher priority blocked tasks is served first.

once another task's get or put request is served, it gets the highest priority blocked task from the block queue to serve.