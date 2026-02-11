# ELEC2645 - Event-Driven Programming with External Interrupts

Learn how to create responsive, event-driven applications on microcontrollers using external interrupts (EXTI). This lab explores interrupt handling, debouncing strategies, and direct LED brightness control via PWM.

The most important file is [Core/Src/main.c](Core/Src/main.c) which contains the main application logic and interrupt handler.

## The Project

This activity demonstrates **event-driven architecture** using hardware interrupts to respond immediately to button presses without polling. The program uses two buttons to control:

- **BTN2** - Controls external LED brightness (toggles between high and low PWM levels)
- **BTN3** - Toggles LCD display mode between normal and inverse, and controls onboard LD2

### Key Concepts

- **External Interrupts (EXTI)** - GPIO pins that trigger immediate hardware interrupts when their state changes
- **Interrupt Handlers** - Functions (callbacks) that execute in response to interrupt events
- **Debouncing** - Software technique to filter out button noise and prevent multiple unintended triggers from a single press
- **Non-blocking Event-Driven Design** - The main loop doesn't need to continuously check button state; interrupts do all the work
- **PWM-based LED Control** - Uses Timer 4 PWM to control LED brightness levels rather than simple on/off control
- **Volatile Variables** - State tracking that may change outside normal program flow (in interrupt handlers)

## Hardware Setup

### Button Connections

| Button | Nucleo Pin | Interrupt Line | Location | Purpose |
|--------|-----|-----------------|---------|---------|
| BTN2 | PC2 | EXTI | Breadboard | External LED brightness control |
| BTN3 | PC3 | EXTI | Joystick | LCD mode toggle and onboard LD2 control |

### LED Output

| LED | Pin | Control Method | Purpose |
|-----|-----|-----------------|---------|
| External LED | PA9 | Timer 4 PWM (CH1) | Main activity LED controlled via PWM |
| Onboard LD2 | PA5 | GPIO digital output | Status indicator for LCD mode |



## How Interrupts Work

### Traditional Polling Approach - simple but limited

```c
while (1) {
    if (button_is_pressed()) {
        do_something();
    }
}
// This wastes CPU checking constantly and is slow to respond
```

### Interrupt-Driven Approach  - much better!

```c
// Main loop does nothing or low-priority tasks
while (1) {
    // Maybe other work here, or sleep
}

// When button is pressed, hardware automatically calls this:
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {
    // Respond immediately!
    if (GPIO_Pin == BTN2_Pin) {
        toggle_led();
    }
}
```

**Advantages:**
- CPU responds immediately to button press (microseconds instead of milliseconds)
- Main loop can perform other tasks or sleep
- Cleaner, more responsive applications
- Lower power consumption (CPU can idle)

## Debouncing Explained

### The Problem

Mechanical buttons generate **electrical noise** when first pressed or released:

```
Button Press (Ideal):       Reality:
_____|‾‾‾‾‾‾‾              _|‾|_|||‾‾‾‾
                             ↑
                        Multiple interrupts!
```

Without debouncing, a single button press can trigger the interrupt 50+ times in a few milliseconds.

### The Solution: Software Debouncing

Check the time since the last interrupt. Ignore new interrupts that occur too quickly:

```c
static uint32_t last_interrupt_time = 0;
uint32_t current_time = HAL_GetTick();

if ((current_time - last_interrupt_time) > DEBOUNCE_DELAY) {
    // This is a "real" button press, not noise
    last_interrupt_time = current_time;
    toggle_led();
}
```

The `DEBOUNCE_DELAY` macro controls how aggressive the debouncing is:
- **50 ms** - Quick response, good for clean buttons
- **100 ms** - Balanced approach, works for most buttons
- **200 ms** - Conservative, tolerates noisy buttons

Different buttons may need different values!

## The Current Code

### Interrupt Handler Structure

```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    // Static variables preserve state between calls
    static uint32_t btn2_last_interrupt_time = 0;
    static uint32_t btn3_last_interrupt_time = 0;
    uint32_t current_time = HAL_GetTick();
    
    // Check which button triggered the interrupt
    if (GPIO_Pin == BTN2_Pin) {
        // Software debouncing
        if ((current_time - btn2_last_interrupt_time) > DEBOUNCE_DELAY) {
            btn2_last_interrupt_time = current_time;
            
            // Toggle LED brightness state
            led_state = !led_state;
            
            // Set PWM duty cycle
            if (led_state) {
                PWM_SetDuty(&pwm_cfg, 100);  // Full brightness
            } else {
                PWM_SetDuty(&pwm_cfg, 0);   // Off
            }
        }
    }
}
```

### State Tracking

Two volatile variables track the current state:

```c
volatile uint8_t led_state = 0;   // 0 = low brightness, 1 = high brightness
volatile uint8_t lcd_mode = 0;    // 0 = normal, 1 = inverse
```

These are `volatile` because they can change unexpectedly (in interrupt handlers), so the compiler won't optimize them away.

## The Assignment: Debounce Experiments

Your task is to investigate how debouncing affects the button response. You will:

### Part 1: Experiment with Debounce Times

1. **Adjust `DEBOUNCE_DELAY`** - Try these values:
   - 50 ms
   - 100 ms
   - 150 ms
   - 200 ms
   - 300 ms

2. **For each value, observe:**
   - How responsive does the button feel?
   - Do you get any multiple toggles from a single press?
   - At what point does the button feel sluggish?

### Part 2: LED Brightness Control

The current code alternates the LED between:
- **100% brightness** (full on)
- **0% brightness** (completely off)

**Modify the code** to instead toggle between:
- **100% brightness** (full on)
- **50% brightness** (dimmed)

This demonstrates that PWM gives you **fine-grained control** over LED intensity, not just binary on/off.

### Part 3: Optional Challenges

- **Cycle through multiple brightness levels** - Instead of toggling between 2 levels, cycle through 4-5 levels (0%, 25%, 50%, 75%, 100%)
- **Add serial output** - Print to UART when buttons are pressed and the debounce status
- **Measure actual debounce effectiveness** - Count how many times interrupts would have fired without debouncing
- **Test both buttons** - See if BTN3 needs a different debounce delay than BTN2

## Running the Code

When you run the program:

1. **Initialization** - System clock, peripherals, and inputs are configured
2. **LCD Display** - Shows startup message on the LCD screen
3. **Main Loop** - Waits for interrupts (button presses)
4. **Button Presses** - Each button press immediately triggers its interrupt handler

The program is **event-driven**: the main loop doesn't do anything. All the action happens in the `HAL_GPIO_EXTI_Callback()` function when buttons are pressed.

## Setup Instructions

### Prerequisites

1. **Completed previous labs**
   - Blinky to understand GPIO and basic LED control
   - PWM lab to understand brightness control
   - Understanding of timers and PWM signals

2. **Configure the Project**
   - Open this folder in VS Code
   - When prompted "*Would you like to configure discovered CMake project as STM32Cube project*", click **Yes**
   - Allow the STM32 extension to complete initialization
   - Select **Debug** configuration when prompted

3. **Verify Hardware Connection**
   - Connect the Nucleo board via USB
   - Check that the board appears under "STM32CUBE Devices and Boards" in the Run and Debug sidebar
   - Ensure BTN2 and BTN3 are properly connected

4. **Build and Run**
   - Click **Build** in the bottom status bar to verify compilation
   - Open Run and Debug panel (`Ctrl+Shift+D`)
   - Select **"STM32Cube: STLink GDB Server"** and click Run
   - Press **F5** or the play button to start debugging

## Key Code Sections

### PWM Configuration

```c
PWM_cfg_t pwm_cfg = {
    .htim = &htim4,
    .channel = TIM_CHANNEL_1,
    .tick_freq_hz = 1000000,
    .min_freq_hz = 10,
    .max_freq_hz = 50000,
    .setup_done = 0
};
```

### Debounce Definition

```c
#define DEBOUNCE_DELAY 200  // milliseconds - MODIFY THIS FOR YOUR EXPERIMENTS
```

### PWM Library Functions

```c
// Initialize PWM with configuration
PWM_Init(&pwm_cfg);

// Set frequency (in Hz)
PWM_SetFreq(&pwm_cfg, 1000);

// Set brightness as duty cycle (0-100%)
PWM_SetDuty(&pwm_cfg, 50);    // 50% brightness
PWM_SetDuty(&pwm_cfg, 100);   // Full brightness
PWM_SetDuty(&pwm_cfg, 0);     // Off

// Stop PWM output
PWM_Off(&pwm_cfg);
```

## Troubleshooting

### Button presses not detected
- Check that EXTI (External Interrupt) is configured for the button pins in STM32CubeMX
- Verify the GPIO pins in main.h match your hardware
- Check that GPIO_Init() is called before waiting for interrupts

### Multiple toggles from single press
- Your `DEBOUNCE_DELAY` is too short
- Increase it in 50ms increments until the problem goes away
- Note: Very slow buttons may need 300-500ms delays

### LED doesn't change brightness
- Verify PWM_Init() is called in main()
- Check that Timer 4 Channel 1 is configured correctly
- Confirm the LED is connected to PA9
- Try PWM_SetDuty() with 100 to verify PWM is working

