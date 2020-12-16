# STM32-UART-Example

- Uses STM32CubeMX & HAL
- Bare metal (no RTOS)
- Set up UART with circular DMA channel

## main.h

```
/* USER CODE BEGIN Includes */
#include <string.h>
#include <stdio.h>
/* USER CODE END Includes */
```

```
/* USER CODE BEGIN Private defines */
#define RXDMABUFFERSIZE 100					// Size of UART RX DMA circular buffer.
#define CMDBUFSIZE 50						// Size of command buffer for use in main thread.
uint8_t uartRxDmaBuffer[RXDMABUFFERSIZE];	// UART RX DMA circular buffer. (Private)
uint8_t uartRxIdleTimeout;					// Signal to main thread that commandBuffer contains data to be consumed. (Public)
uint8_t uartRxCommandBuffer[CMDBUFSIZE];	// UART idle interrupt will copy from DMA circular buffer into this buffer. Could contain multiple commands if main thread isn't servicing quick enough. (Public)
uint32_t uartRxDmaStartPoint;				// Tracks the circular DMA buffer, used in idle interrupt. (Private)
/* USER CODE END Private defines */
```

## stm32g4xxx_it.c

```
/**
  * @brief This function handles LPUART1 global interrupt.
  */
void LPUART1_IRQHandler(void)
{
  /* USER CODE BEGIN LPUART1_IRQn 0 */
	if (__HAL_UART_GET_IT_SOURCE(&hlpuart1, UART_IT_IDLE) && __HAL_UART_GET_FLAG(&hlpuart1, UART_FLAG_IDLE))
	{
		__HAL_UART_CLEAR_IDLEFLAG(&hlpuart1);

		uint32_t nextStartPoint = RXDMABUFFERSIZE - __HAL_DMA_GET_COUNTER(&hdma_lpuart1_rx);
		if(uartRxDmaStartPoint == nextStartPoint || uartRxIdleTimeout)
		{	//No bytes received OR main thread hasn't serviced the uartRxIdleTimeout signal
		}
		else
		{	//Bytes received, copy bytes into commandBuffer and signal to main thread that it's ready to use.
		  uint32_t endPoint =  nextStartPoint;
		  if(endPoint == 0)						// Handle special case where end point is the very last byte of the DMA buffer
			  endPoint = RXDMABUFFERSIZE - 1;
		  else
			 endPoint = nextStartPoint - 1;

		  memset(uartRxCommandBuffer, 0, CMDBUFSIZE);

		  if(endPoint > uartRxDmaStartPoint)
		  {
			  memcpy(uartRxCommandBuffer, uartRxDmaBuffer + uartRxDmaStartPoint, endPoint - uartRxDmaStartPoint + 1);
		  }
		  else
		  {
			  memcpy(uartRxCommandBuffer, uartRxDmaBuffer + uartRxDmaStartPoint, RXDMABUFFERSIZE - uartRxDmaStartPoint);
			  memcpy(uartRxCommandBuffer + (RXDMABUFFERSIZE - uartRxDmaStartPoint), uartRxDmaBuffer, endPoint);
		  }
		  uartRxDmaStartPoint = nextStartPoint;
		  uartRxIdleTimeout = 1;
		}
	}
  /* USER CODE END LPUART1_IRQn 0 */
  HAL_UART_IRQHandler(&hlpuart1);
  /* USER CODE BEGIN LPUART1_IRQn 1 */

  /* USER CODE END LPUART1_IRQn 1 */
}
```

## main.c

```
	  uint32_t value;
	  if(uartRxIdleTimeout)
	  {
		  sscanf((char*)uartRxCommandBuffer, "%u", &value);
		  uartRxIdleTimeout = 0;
	  }
```
