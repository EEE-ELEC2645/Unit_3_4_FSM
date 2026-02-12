# ELEC2645 - Finite State Machine (FSM) Demo

Learn about **Finite State Machines (FSM)** - a fundamental programming concept used in embedded systems, games, user interfaces, and control systems. This interactive demo shows how the same input can produce different outputs depending on the system's current state.

The most important file is [Core/Src/main.c](Core/Src/main.c) which contains the FSM implementation and state handlers.

---

## What is a Finite State Machine?

A **Finite State Machine** is a programming pattern where:

1. **The system has a set number of "states"** (modes of operation)
2. **The system is in exactly ONE state at any time**
3. **Inputs are handled differently depending on which state the system is in**
4. **External events can cause transitions between states**

### Real-World Examples

- **Traffic Lights**: States are RED, YELLOW, GREEN - timers cause transitions
- **Mobile Phone**: States are LOCKED, HOME_SCREEN, IN_CALL, CAMERA - button presses cause transitions
- **Washing Machine**: States are IDLE, WASH, RINSE, SPIN - sensor readings cause transitions
- **Game Character**: States are IDLE, WALKING, JUMPING, ATTACKING - player input causes transitions

### The Key Insight

**The SAME input produces DIFFERENT behavior depending on the current state!**

Example: Pressing "A" on a game controller:
- In MENU state → selects menu item
- In GAMEPLAY state → character jumps
- In PAUSE state → resumes game

---

## This Demo Project

This activity demonstrates FSM concepts using the Nucleo,  joystick and LCD display.

### The Four States

| State | What It Does | Joystick Behavior |
|-------|--------------|-------------------|
| **CIRCLES** | Draw colored circles | X-axis changes color, Y-axis changes size |
| **RECTANGLES** | Move a rectangle | Joystick position moves rectangle around screen |
| **LINES** | Draw directional lines | Joystick direction draws line from center |
| **SPRITES** | Move a sprite character | 8-way discrete movement (N, NE, E, SE, S, SW, W, NW) |

### State Transitions

**Pressing the joystick button** triggers a state transition:

```
CIRCLES → RECTANGLES → LINES → SPRITES → CIRCLES → ...
```

The system cycles through all states continuously.

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
The system cycles through all states continuously.

---

## How the FSM Works

### 1. State Variable

The current state is stored in a variable:

```c
volatile FSM_State_t current_state = STATE_CIRCLES;
```

It's `volatile` because it can change in the interrupt handler.

### 2. Main Loop - The FSM Controller

```c
while (1) {
    // Read joystick input (SAME for all states)
    Joystick_Read(&joystick_cfg, &joystick_data);
    
    // Execute DIFFERENT code based on current state
    switch (current_state) {
        case STATE_CIRCLES:
            handle_state_circles(&joystick_data);
            break;
        
        case STATE_RECTANGLES:
            handle_state_rectangles(&joystick_data);
            break;
        
        case STATE_LINES:
            handle_state_lines(&joystick_data);
            break;
        
        case STATE_SPRITES:
            handle_state_sprites(&joystick_data);
            break;
    }
}
```

**This is the core of the FSM pattern**: A switch statement that executes different code based on the current state.

### 3. State Handlers

Each state has its own handler function that processes joystick input:

```c
void handle_state_circles(Joystick_t* joy) {
    // Map joystick X to color
    uint8_t color = (uint8_t)((joy->coord_mapped.x + 1.0f) * 4.0f) + 2;
    
    // Map joystick Y to radius
    uint8_t radius = (uint8_t)((joy->coord_mapped.y + 1.0f) * 15.0f) + 15;
    
    // Draw circle
    LCD_Draw_Circle(120, 120, radius, color, 1);
}
```

### 4. State Transitions

An **interrupt handler** responds to button presses and changes the state:

```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {
    if (GPIO_Pin == BTN3_Pin) {
        // Debouncing check...
        
        // STATE TRANSITION: Move to next state (wraps around)
        // The (FSM_State_t) cast is needed because arithmetic operations
        // return an integer, so we must cast back to enum type
        current_state = (FSM_State_t)((current_state + 1) % STATE_COUNT);
        
        // Set flag to refresh display
        state_changed = 1;
    }
}
```

The `% STATE_COUNT` operation makes the states **wrap around**: after the last state, it goes back to the first.

---

## Hardware Setup

### Required Components

- STM32 Nucleo-L476RG board
- LCD display (ST7789V2)
- Analog joystick module with button
- Breadboard and jumper wires

### Joystick Connections

| Function | Nucleo Pin | ADC Channel |
|----------|------------|-------------|
| X-axis | A5 | ADC1_CH1 |
| Y-axis | A4 | ADC1_CH2 |
| Button (BTN3) | PC3 | EXTI3 |

---

## Key Programming Concepts

### 1. Enumerations (enum)

States are defined using an enum for type safety and readability:

```c
typedef enum {
    STATE_CIRCLES = 0,
    STATE_RECTANGLES,
    STATE_LINES,
    STATE_SPRITES
} FSM_State_t;
```

### 2. Switch Statements

The switch statement is the classic FSM pattern for selecting behavior:

```c
switch (current_state) {
    case STATE_CIRCLES:
        // Circle behavior
        break;
    case STATE_RECTANGLES:
        // Rectangle behavior
        break;
    // etc.
}
```

### 3. Function Pointers (Advanced Alternative)

An alternative FSM implementation uses function pointers:

```c
// Array of function pointers (one per state)
void (*state_handlers[])(Joystick_t*) = {
    handle_state_circles,
    handle_state_rectangles,
    handle_state_lines,
    handle_state_sprites
};

// Call the current state's handler
state_handlers[current_state](&joystick_data);
```

### 4. Interrupt Service Routines (ISR)

ISRs should be kept **short and simple**:

✅ **Do:** Set flags, update state variables, toggle LEDs  
❌ **Don't:** Use `HAL_Delay()`, print to serial, perform complex calculations

---

## The Educational Value

This demo teaches several important concepts:

1. **FSM Design Pattern** - How to structure code around states
2. **State-Dependent Behavior** - Same input, different output
3. **Interrupt-Driven State Transitions** - External events trigger changes
4. **Modular Code Organization** - Each state has its own handler function
5. **Enum Type Safety** - Using enums instead of magic numbers
6. **Real-Time Interaction** - Responsive user interface design

---

## Extending the Demo

Try these modifications to deepen your understanding:

### Add a New State

1. Add a new state to the enum (e.g., `STATE_POLYGONS`)
2. Increment `STATE_COUNT`
3. Add a new state handler function
4. Add the new case to the switch statement
5. Add state name to the `state_names` array

### Add State-Specific LED Patterns

Make each state light the onboard LED differently:

```c
void set_led_for_state(FSM_State_t state) {
    switch(state) {
        case STATE_CIRCLES: PWM_SetDuty(&pwm_cfg, 25); break;
        case STATE_RECTANGLES: PWM_SetDuty(&pwm_cfg, 50); break;
        // etc.
    }
}
```

### Implement Conditional Transitions

Add logic so states only change under certain conditions:

```c
// Only allow transition if joystick is centered
if (joy->magnitude < 0.1f) {
    current_state = next_state;
}
```

---

## Common FSM Patterns

FSMs appear everywhere in embedded systems:
FSMs appear everywhere in embedded systems:

### 1. Menu Systems

```c
enum MenuState { MAIN_MENU, SETTINGS, ABOUT };
// Button A = select, Button B = back
```

### 2. Communication Protocols

```c
enum UARTState { IDLE, RECEIVING, PROCESSING, SENDING };
// State changes on byte received, timeout, etc.
```

### 3. Motor Control

```c
enum MotorState { STOPPED, STARTING, RUNNING, BRAKING, ERROR };
// State changes based on sensor feedback
```

### 4. Safety Systems

```c
enum SafetyState { SAFE, WARNING, ALARM, SHUTDOWN };
// State changes based on sensor thresholds
```

---

## Debouncing in FSM Context

This demo also demonstrates **software debouncing** - essential for reliable button input:

### The Problem

Mechanical buttons generate electrical noise:

```
Button Press (Ideal):       Reality:
_____|‾‾‾‾‾‾‾              _|‾|_|||‾‾‾‾
                             ↑
                        Multiple interrupts!
```

### The Solution

```c
static uint32_t last_interrupt_time = 0;
uint32_t current_time = HAL_GetTick();

if ((current_time - last_interrupt_time) > DEBOUNCE_DELAY) {
    // This is a real button press
    last_interrupt_time = current_time;
    current_state = next_state;
}
```

The `DEBOUNCE_DELAY` (200ms in this demo) filters out noise by ignoring interrupts that occur too quickly.

---

## Running the Code

### Build and Flash

1. Open this folder in VS Code
2. When prompted to configure as STM32Cube project, click **Yes**
3. Select **Debug** configuration
4. Click **Build** in the status bar
5. Connect your Nucleo board via USB
6. Press **F5** to flash and run

### Using the Demo

1. **Watch the startup sequence** - LCD shows "Finite State Machine" splash screen
2. **Press joystick button** - Cycles through states (CIRCLES → RECTANGLES → LINES → SPRITES)
3. **Move joystick** - Each state responds differently to the same input
4. **Observe the pattern** - Same hardware input, different software behavior based on state

---

## Code Structure

### File Organization

```
Core/Src/main.c          - Main FSM logic and state handlers
Core/Inc/main.h          - GPIO pin definitions
Joystick/                - Joystick driver with Direction enum
ST7789V2_Driver_STM32L4/ - LCD display driver
PWM/                     - PWM control library
```

### Main Components

```c
// State definition
typedef enum {
    STATE_CIRCLES,
    STATE_RECTANGLES,
    STATE_LINES,
    STATE_SPRITES
} FSM_State_t;

// State variable
volatile FSM_State_t current_state = STATE_CIRCLES;

// Main loop - FSM controller
while (1) {
    Joystick_Read(&joystick_cfg, &joystick_data);
    
    switch (current_state) {
        case STATE_CIRCLES:
            handle_state_circles(&joystick_data);
            break;
        // ... other states
    }
}

// Interrupt - state transition
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {
    if (GPIO_Pin == BTN3_Pin) {
        // Debouncing...
        current_state = (FSM_State_t)((current_state + 1) % STATE_COUNT);
    }
}
```

---

## Learning Outcomes

After completing this activity, you should understand:

1. ✅ **What a Finite State Machine is** and why it's useful
2. ✅ **How to implement an FSM** using enums and switch statements
3. ✅ **State-dependent behavior** - same input, different output
4. ✅ **Interrupt-driven transitions** - external events change state
5. ✅ **Code organization** - modular state handler functions
6. ✅ **Debouncing techniques** - filtering mechanical button noise
7. ✅ **Enum type casting** - why `(FSM_State_t)` is needed in arithmetic

---

## Further Reading

- **State Pattern** - Object-oriented approach to FSM (design patterns)
- **Mealy vs Moore Machines** - Two types of FSM (theory of computation)
- **State Transition Diagrams** - Visual representation of FSM behavior
- **Hierarchical State Machines** - Complex FSMs with nested states
- **UML State Diagrams** - Industry-standard FSM notation

---

## License

This project is part of the ELEC2645 Embedded Systems module at the University of Leeds.
````