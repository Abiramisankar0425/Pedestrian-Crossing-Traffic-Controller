# Pedestrian-Crossing-Traffic-Controller
An interactive, zero-latency traffic light controller implemented on a PIC18F4580 microcontroller using external hardware interrupts (INT0) and a modular state machine.
# PIC18F4580 Pedestrian Crossing Traffic Controller

## 💡 Overview

The **Pedestrian Crossing Traffic Controller** is an interrupt-driven embedded systems project built using the **PIC18F4580 microcontroller**.

The system simulates a real-world traffic signal controller where pedestrian crossing requests are handled using the **INT0 External Interrupt**. Instead of continuously polling a push button, the controller instantly detects pedestrian requests through hardware interrupts, ensuring responsive and safe traffic light transitions.

The project demonstrates:

* External Interrupts (INT0)
* State Machine Design
* Traffic Signal Control
* Pedestrian Crossing Logic
* Embedded Event Handling
* GPIO Control
* Interrupt Service Routines (ISR)

---

## 🎯 Project Objective

Modern traffic intersections must respond immediately to pedestrian crossing requests while maintaining safe vehicle flow.

This project implements:

* Normal vehicle traffic operation
* Interrupt-driven pedestrian request detection
* Safe traffic light transitions
* Walk signal timing
* Pedestrian warning strobe
* Return to normal traffic flow

---

## 🚦 Features

### Vehicle Traffic Control

* Green Light
* Yellow Warning Light
* Red Stop Light

### Pedestrian Signals

* Walk Signal
* Don't Walk Signal

### Interrupt-Based Request Handling

* Uses INT0 external interrupt
* Zero-latency request detection
* Responsive pedestrian crossing system

### Safe State Machine

* Green → Yellow → Red → Walk → Warning Flash → Normal

---

## 🛠️ Hardware Requirements

* PIC18F4580 Microcontroller
* Push Button Switch
* 5 LEDs
* 220Ω Resistors (5)
* 10kΩ Pull-Up Resistor
* Breadboard
* Connecting Wires
* 5V Power Supply
* MPLAB X IDE
* XC8 Compiler
* Proteus Design Suite (Optional)

---

## 🔌 Hardware Connections

### Pedestrian Request Button

| PIC Pin    | Pin Number | Connection  |
| ---------- | ---------- | ----------- |
| RB0 / INT0 | 33         | Push Button |

Connection Details:

* One terminal → GND
* Other terminal → RB0
* 10kΩ Pull-up resistor between RB0 and +5V

---

### Vehicle Traffic LEDs

| PIC Pin | LED        |
| ------- | ---------- |
| RD0     | Car Green  |
| RD1     | Car Yellow |
| RD2     | Car Red    |

---

### Pedestrian LEDs

| PIC Pin | LED        |
| ------- | ---------- |
| RD3     | Walk       |
| RD4     | Don't Walk |

---

## 🗺️ Circuit Diagram

### Proteus Schematic

Add your schematic image here:

```markdown
![Pedestrian Crossing Controller](images/schematic.png)
```

---

## ⚙️ Traffic Signal States

### State 1 – Normal Traffic

Vehicle:

* Green ON

Pedestrian:

* Don't Walk ON

Output:

```text
RD0 = 1
RD4 = 1
```

---

### State 2 – Yellow Warning

Vehicle:

* Yellow ON

Pedestrian:

* Don't Walk ON

Output:

```text
RD1 = 1
RD4 = 1
```

---

### State 3 – Pedestrian Walk

Vehicle:

* Red ON

Pedestrian:

* Walk ON

Output:

```text
RD2 = 1
RD3 = 1
```

---

### State 4 – Walk Ending Warning

Vehicle:

* Red ON

Pedestrian:

* Walk and Don't Walk alternate

Output:

```text
RD2 = 1
RD3 ↔ RD4
```

---

### State 5 – Return to Normal

Vehicle:

* Green ON

Pedestrian:

* Don't Walk ON

Output:

```text
RD0 = 1
RD4 = 1
```

---

## 🔄 System Flow

```text
START
  |
  V
Normal Traffic
(Car Green)
  |
  V
Pedestrian Button Press
(INT0 Interrupt)
  |
  V
Request Flag Set
  |
  V
Yellow Warning
  |
  V
Car Red
Pedestrian Walk
  |
  V
Walk Warning Flash
  |
  V
Return To Normal Traffic
```

---

## 🧠 Interrupt Architecture

### ISR Responsibilities

The ISR performs only two tasks:

1. Detect INT0 trigger
2. Set pedestrian request flag

```c
pedestrian_request = 1;
```

### Why?

Heavy processing and delays should never be placed inside an ISR because:

* Interrupts must execute quickly
* Prevents system lockups
* Improves responsiveness
* Follows industry best practices

---

## 💻 Complete XC8 Source Code

```c
#include <xc.h>

// Configuration Bits (20MHz crystal oscillator, watchdog timer off, low voltage programming off)
#pragma config OSC = HS
#pragma config WDT = OFF
#pragma config LVP = OFF

// Function prototype for software delay
void delay();
void run_normal_traffic();
void run_pedestrian_sequence();

// Volatile flag representing pedestrian cross request
volatile unsigned char pedestrian_request = 0;

void main(void) {
    // 1. Configure Port Registers
    TRISD = 0x00;  // Configure PORTD as outputs (traffic and pedestrian LEDs)
    TRISB = 0xFF;  // Configure PORTB as inputs (RB0 is Pedestrian button input)
    ADCON1 = 0x0F; // Configure all pins as digital GPIO

    // 2. Configure Weak Pull-ups on PORTB
    RBPU = 0;      // Enable PORTB weak pull-up resistors (Active Low input safety)

    // 3. Configure External Interrupt (INT0 on RB0)
    INTEDG0 = 0;   // Trigger on Falling Edge (High-to-Low voltage on button press)
    INT0IF = 0;    // Clear interrupt flag initially
    INT0IE = 1;    // Enable INT0 external interrupt
    PEIE = 1;      // Enable peripheral interrupts
    GIE = 1;       // Enable global interrupts

    // Initial Safe State
    run_normal_traffic();

    while (1) {
        if (pedestrian_request == 1) {
            // Execute safe pedestrian walk transition
            run_pedestrian_sequence();
            
            // Clear request flag and resume normal vehicle flow
            pedestrian_request = 0; 
        } else {
            run_normal_traffic();
        }
    }
}

// 4. State Machine Helper Functions
void run_normal_traffic() {
    // Car Green (RD0 = 1), Pedestrian Don't Walk (RD4 = 1) -> 0x11 (0001 0001)
    LATD = 0x11;
}

void run_pedestrian_sequence() {
    // Phase 1: Car Yellow warning (RD1 = 1), Pedestrian Don't Walk (RD4 = 1) -> 0x12 (0001 0010)
    LATD = 0x12;
    delay();

    // Phase 2: Car Red stop (RD2 = 1), Pedestrian Walk (RD3 = 1) -> 0x0C (0000 1100)
    LATD = 0x0C;
    delay();

    // Phase 3: Walk warning strobe (Car Red RD2 remains ON while walk signals alternate)
    // Alternate between Walk (RD3) and Don't Walk (RD4)
    for (int k = 0; k < 3; k++) {
        LATD = 0x14; // Car Red (RD2=1), Pedestrian Don't Walk (RD4=1) -> 0x14 (0001 0100)
        delay();
        LATD = 0x0C; // Car Red (RD2=1), Pedestrian Walk (RD3=1) -> 0x0C (0000 1100)
        delay();
    }

    // Phase 4: All Stop (Car Red RD2=1, Pedestrian Don't Walk RD4=1) -> 0x14 (0001 0100)
    LATD = 0x14;
    delay();
}

// 5. Interrupt Service Routine (ISR)
void __interrupt() isr(void) {
    // Verify that the external interrupt INT0 caused the trigger
    if (INT0IF == 1) {
        // Record walk request
        pedestrian_request = 1; 
        
        // Clear flag in software to enable future interrupts
        INT0IF = 0; 
    }
}

// Software delay definition
void delay() {
    int i, j;
    for (i = 0; i < 600; i++) {
        for (j = 0; j < 600; j++) {
            // Delay loop
        }
    }
}
```

---

## 🪜 How to Build & Run

### 1️⃣ Create MPLAB X Project

* Open MPLAB X IDE
* File → New Project
* Standalone Project

### 2️⃣ Select Device

Choose:

```text
PIC18F4580
```

### 3️⃣ Select Compiler

Choose:

```text
XC8 Compiler
```

### 4️⃣ Add Source Code

Create:

```text
main.c
```

Paste the code.

### 5️⃣ Build Project

Click:

```text
Clean and Build Project
```

### 6️⃣ Generate HEX File

The compiler generates:

```text
main.hex
```

### 7️⃣ Load Into Proteus

Attach HEX file to PIC18F4580.

### 8️⃣ Run Simulation

Observe:

* Car Green LED ON
* Pedestrian Red LED ON

Press button:

* Yellow Warning
* Car Red
* Pedestrian Walk
* Flash Warning
* Return to Normal

---

## 📊 Expected Output

### Idle

```text
Car Green ON
Pedestrian Red ON
```

### Button Pressed

```text
Yellow Warning
```

### Crossing Phase

```text
Car Red ON
Pedestrian Walk ON
```

### Walk Ending

```text
Walk Flashing
Don't Walk Flashing
```

### Return

```text
Car Green ON
Pedestrian Red ON
```

---

## 🏙️ Real-World Applications

* Traffic Signal Controllers
* Smart City Infrastructure
* Pedestrian Crosswalk Systems
* Road Safety Systems
* Embedded Systems Education
* Interrupt-Based Control Systems

---

## ✅ Advantages

* Fast Interrupt Response
* Clean Embedded Architecture
* Low CPU Overhead
* Industry-Style State Machine
* Simple Hardware Design
* Expandable for Smart Traffic Systems

---

## ⚠️ Limitations

* Uses Software Delays
* No Timer-Based Scheduling
* No Multiple Pedestrian Requests Queue
* No Vehicle Detection Sensors

---

## 🔮 Future Enhancements

* Timer-Based State Control
* Debouncing Logic
* Multiple Crossing Requests Queue
* Traffic Density Sensors
* Smart Intersection Networking
* CAN Bus Integration
* Emergency Vehicle Priority


---


## 🙏 Acknowledgments

Designed and simulated using:

* PIC18F4580 Microcontroller
* MPLAB X IDE
* XC8 Compiler
* Proteus Design Suite

---

## 📚 Resources

* PIC18F4580 Datasheet
* MPLAB X IDE
* XC8 Compiler
* Proteus Design Suite
* Embedded Systems Programming
* Traffic Signal Control Fundamentals
