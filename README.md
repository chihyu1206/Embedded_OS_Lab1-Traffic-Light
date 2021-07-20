# Embedded_OS_Lab1-Traffic-Light
NCKU Analysis and Implementation of Embedded Operating Systems Lab 1

## Lab Requirement

* Must use MultiTask, e.g.  One for LED ,  other for Button.
  * Inter process communication (IPC).
* LED have two state
  * The green LED lights up for 5 seconds, then turns to a red LED lights up for 5 seconds (green LED off), and then switches back to the green LED lights up for 5 seconds (red LED off), and so on.
  * The red LED blinking.
* If the button is pressed, the LED will switch to another state.
  * Debounce.
  * Edge detection.

## Environment

* OS: Ubuntu 18.04
* Development Kit: [STM32F407G-DISC1](https://www.st.com/en/evaluation-tools/stm32f4discovery.html)
* [FreeRTOS v10.2.1](https://github.com/FreeRTOS/FreeRTOS/tree/V10.2.1)
* [STM32CubeIDE-DEB 1.5.1](https://www.st.com/en/development-tools/stm32cubeide.html)

## Concepts

2個LED Task分別代表2個states，1個Button Task控制LED要在哪個state。
使用`uxPriority(定義這個任務的優先權，在FreeRTOS中0最低)`將Button Task的優先權設定得比LED Task高。
```c=
  // will be used to create the task
  void ButtonTask(void *);
  void LEDTask1(void *);
  void LEDTask2(void *);
  
  
  QueueHandle_t xQueue;
  TaskHandle_t xHandle1, xHandle2;
  
  xQueue = xQueueCreate(1, sizeof(int));
  xTaskCreate(ButtonTask, "ButtonTask", 1024, (void *)1, 2, NULL);
  xTaskCreate(LEDTask1, "LEDTask1", 1024, (void *)1, 1, &xHandle1);
  xTaskCreate(LEDTask2, "LEDTask2", 1024, (void *)1, 1, &xHandle2);
  vTaskStartScheduler();
```

使用`GPIO_PIN_SET`和`GPIO_PIN_RESET`控制亮暗，並用`vTaskDelay`控制2個燈切換間隔的時間(ms)。
```c=
void LEDTask1(void *pvParameters) {
	int flag = 0;
    while (1) {
        xQueueReceive(xQueue, &flag, 0);
        if (flag == 0) {
 		   HAL_GPIO_WritePin(Green_LED_GPIO_Port, GPIO_PIN_12, GPIO_PIN_SET);
 		   vTaskDelay(5000);
 		   HAL_GPIO_WritePin(Green_LED_GPIO_Port, GPIO_PIN_12, GPIO_PIN_RESET);
 		   
           HAL_GPIO_WritePin(Red_LED_GPIO_Port, GPIO_PIN_14, GPIO_PIN_SET);
 		   vTaskDelay(5000);
 		   HAL_GPIO_WritePin(Red_LED_GPIO_Port, GPIO_PIN_14, GPIO_PIN_RESET);
        }
    }
    return;
}
```

每隔1000 ms(1 sec)，讓紅燈切換亮暗。
```c=
void LEDTask2(void *pvParameters) {
	int flag = 0;
	while (1) {
		xQueueReceive(xQueue, &flag, 0);
		if (flag == 1) {
            HAL_GPIO_WritePin(Red_LED_GPIO_Port, GPIO_PIN_14, GPIO_PIN_SET);
			vTaskDelay(1000);
			HAL_GPIO_WritePin(Red_LED_GPIO_Port, GPIO_PIN_14, GPIO_PIN_RESET);
			vTaskDelay(1000);
		}
	}
    return;
}
```
使用`HAL_GPIO_ReadPin`來判斷是否有按按鈕。
用`taskNum`來控制要執行哪個state {0: LEDTask 1, 1: LEDTask 2}，當要恢復`vTaskResume`其中1個state前，要先用`vTaskSuspend`將另個state暫停。
切換後會將`taskNum`用`xQueueOverwrite`寫入queue，讓其中一個state得以執行。
```c=
void ButtonTask(void *pvParameters) {

	int taskNum = 0;

	while (1) {
		while (!HAL_GPIO_ReadPin(Blue_Button_GPIO_Port, GPIO_PIN_0)) {
			vTaskDelay(5);
		}
		vTaskDelay(100);
		if (HAL_GPIO_ReadPin(Blue_Button_GPIO_Port, GPIO_PIN_0)) {
			if (taskNum == 0) {
				taskNum = 1;
				vTaskSuspend(xHandle1);
				HAL_GPIO_WritePin(Green_LED_GPIO_Port, GPIO_PIN_12, GPIO_PIN_RESET);
				HAL_GPIO_WritePin(Red_LED_GPIO_Port, GPIO_PIN_14, GPIO_PIN_RESET);
				vTaskResume(xHandle2);
			} else {
				taskNum = 0;
				vTaskSuspend(xHandle2);
				HAL_GPIO_WritePin(Green_LED_GPIO_Port, GPIO_PIN_12, GPIO_PIN_RESET);
				HAL_GPIO_WritePin(Red_LED_GPIO_Port, GPIO_PIN_14, GPIO_PIN_RESET);
				vTaskResume(xHandle1);
			}
			xQueueOverwrite(xQueue, &taskNum);
		}
		vTaskDelay(100);
	}
    return;
}
```

[結果影片](https://imgur.com/8X6XUwc)
