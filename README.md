# STM32L432KC External Interrupt & Latency Measurement

A push button on **PB3** raises an external interrupt (EXTI line 3), and the interrupt handler toggles an LED on **PB4**. Interrupt latency is measured on an oscilloscope by probing the button and LED signals.

## Overview

The button reads by polling it inside the `while(1)` loop. Here the CPU does not poll the pin. The hardware detects the edge on PB3 and jumps to an interrupt service routine (ISR), so the response time depends on the interrupt path rather than on where the main loop happens to be.

This repo covers:

- Configuring a GPIO pin as an external interrupt (`GPIO_EXTI3`) in STM32CubeMX
- Enabling the EXTI line 3 interrupt in the NVIC
- Writing the ISR in `stm32l4xx_it.c`
- Measuring the delay between the button edge and the LED response with an oscilloscope
- Observing and explaining button contact bounce

## Hardware

| Component | Details |
|---|---|
| Board | NUCLEO-L432KC (STM32L432KCU, ARM Cortex-M4) |
| Input | Push button from 3V3 to **D13 (PB3)**, with a ~1 kΩ pull-down to GND |
| Output | LED on **D12 (PB4)** through a ~330 Ω resistor to GND |
| Instrument | Oscilloscope, Channel 1 on the button and Channel 2 on the LED |

Wiring follows Figure of the lab handout. Pressing the button connects 3V3 to PB3, so a press produces a rising edge on the pin.

## Configuration (STM32CubeIDE / CubeMX)

| Setting | Value |
|---|---|
| PB3 | `GPIO_EXTI3` (external interrupt) |
| PB4 | `GPIO_Output`, push-pull, no pull-up/pull-down, low speed |
| NVIC | `EXTI line3 interrupt` enabled, preemption priority 0 |
| Board | NUCLEO-L432KC (selected through the Board Selector) |

## The Interrupt Handler

In `Core/Src/stm32l4xx_it.c`:

```c
void EXTI3_IRQHandler(void)
{
  /* USER CODE BEGIN EXTI3_IRQn 0 */
  HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_4);   // toggle the LED
  /* USER CODE END EXTI3_IRQn 0 */

  HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_3);    // HAL clears the pending flag

  /* USER CODE BEGIN EXTI3_IRQn 1 */

  /* USER CODE END EXTI3_IRQn 1 */
}
```

The LED is toggled before the HAL handler runs, which keeps the path from edge to output short.

## Latency Measurement

Cursors on the oscilloscope capture give two intervals:

- **C0**: time from the button edge to the LED edge (total interrupt latency)
- **C1**: duration of the LED pulse, which spans the cursors that mark the handler's execution

| Measurement | Value |
|---|---|
| C0 (button edge to LED edge) | _fill in from your scope capture_ |
| C1 | _fill in from your scope capture_ |

Add your oscilloscope screenshot to the repo and link it here:

`![Interrupt latency capture](docs/scope_capture.png)`

## Observations: Contact Bounce

A mechanical button does not make a single clean transition. The contacts bounce for a short time, so one press can produce several edges, and each edge can fire the interrupt. The LED may then toggle more than once per press and end up in the wrong state.

Ways to avoid it:

- **Hardware:** an RC low-pass filter on the button line, or a Schmitt-trigger input to clean up the slow, noisy edge (the STM32 GPIO input stage already includes a Schmitt trigger)
- **Software:** ignore further interrupts for a short window after the first edge, for example by comparing `HAL_GetTick()` against the time of the last accepted press

## Repository Contents

- `Core/Src/main.c`: application setup and main loop
- `Core/Src/stm32l4xx_it.c`: EXTI3 interrupt handler
- `*.ioc`: STM32CubeMX configuration
- `docs/`: oscilloscope capture and circuit photos

## Build and Run

1. Open the project in STM32CubeIDE.
2. Wire the circuit as described above.
3. Build the project (**Project > Build Project**).
4. Flash it with **Run > Debug**, then press **Resume (F8)**, since the debugger halts at the start of `main()` by default.
5. Press the button and watch the LED on the board and on the scope.

## Author

Reshma Merin Thomas, CPE 187 Embedded Systems Design
