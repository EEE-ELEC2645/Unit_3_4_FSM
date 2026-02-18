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

This activity demonstrates FSM concepts using the Nucleo, joystick and LCD display.

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

---

## How the FSM Works

### 1. State Variable

The current state is stored in a global variable:

```c
volatile FSM_State_t g_current_state = STATE_CIRCLES;
```

It's named with the `g_` prefix to indicate it's a global variable accessed across multiple scopes (main loop and interrupt handler). It's `volatile` because it can change in the interrupt handler.

### 2. Main Loop - The FSM Controller

```c
while (1) {
    // Read joystick input (SAME for all states)
    Joystick_Read(&joystick_cfg, &joystick_data);
    
    // Execute DIFFERENT code based on current state
    switch (g_current_state) {
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
        g_current_state = (FSM_State_t)((g_current_state + 1) % STATE_COUNT);
        
        // Set flag to refresh display
        state_changed = 1;
    }
}
```

The `% STATE_COUNT` operation makes the states **wrap around**: after the last state, it goes back to the first.

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
switch (g_current_state) {
    case STATE_CIRCLES:
        // Circle behavior
        break;
    case STATE_RECTANGLES:
        // Rectangle behavior
        break;
    // etc.
}
```

### 3. Interrupt Service Routines (ISR)

ISRs should be kept **short and simple**:

**Do:** Set flags, update state variables, toggle LEDs  
**Don't:** Use `HAL_Delay()`, print to serial, perform complex calculations

---

## Your Task - Extending the Demo

Try these modifications to deepen your understanding:

### Add a New State

1. Add a new state to the enum (e.g., `STATE_POLYGONS`)
2. Increment `STATE_COUNT`
3. Add a new state handler function
4. Add the new case to the switch statement
5. Add state name to the `state_names` array

### Add State-Specific LED Patterns

Make each state light the breadboard LED differently:

```c
void set_led_for_state(void) {
    switch(g_current_state) {
        case STATE_CIRCLES: PWM_SetDuty(&pwm_cfg, 25); break;
        case STATE_RECTANGLES: PWM_SetDuty(&pwm_cfg, 50); break;
        // etc.
    }
}
```

### Optional - Implement Conditional Transitions

Add logic so states only change under certain conditions:

```c
// Only allow transition if joystick is centered
if (joy->magnitude < 0.1f) {
    g_current_state = next_state;
}
```

---

## Common FSM Patterns

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