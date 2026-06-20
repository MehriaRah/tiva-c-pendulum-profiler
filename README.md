## 📖 Project Overview

This repository contains the complete firmware for an event-driven spatial tracker. By leveraging the **TI Tiva C Series** microcontroller, the system monitors a physical pendulum’s motion in real-time, fires precise ultrasonic sonar waves to gauge distance, and maps the tracking metrics across an 8-bit LED display ribbon. 

Unlike traditional microprocessing loops that constantly burn CPU cycles waiting on sensors, this system operates on a **Pure Background Architecture** driven exclusively by hardware edge transitions and internal stopwatch timers.

---

## 🛠️ Hardware Interfacing Matrix

To set up the lab testbench, interface your Tiva C launchpad with the external peripherals using the following direct pin configurations:

| Component | Device Pin | Connection Direction | Tiva C Pin | Functional Profile |
| :--- | :--- | :---: | :--- | :--- |
| **Pendulum Setup** | Reversal Sensor | `Input` ──► | **`PK0`** | Triggers interrupt on pendulum direction shift |
| **Ultrasonic Sensor** | `Trig` (Trigger) | `Output` ◄── | **`PD0`** | Receives precise 10µs pulses to launch sonic burst |
| **Ultrasonic Sensor** | `Echo` (Return) | `Input` ──► | **`PD1`** | Captures time-of-flight pulse width via edge transitions |
| **Display Ribbon** | 8-LED Array Pins | `Output` ◄── | **`PM0 - PM7`** | Renders dynamic distance as a progressive bar graph |

> ⚠️ **IMPORTANT ELECTRICAL SAFETY NOTE:** The HC-SR04 ultrasonic sensor operates on a **5V** logic level. Because the Tiva C microcontroller input registers natively read at **3.3V**, you should route the `Echo` line into a simple two-resistor voltage divider (e.g., $1\text{ k}\Omega$ and $2\text{ k}\Omega$) before connecting it to **`PD1`** to prevent overvoltage degradation to your launchpad hardware.

---

## 🧠 Core System Mechanics

The entire lifecycle of the application runs asynchronously in the background. The firmware utilizes the Tiva's **Nested Vectored Interrupt Controller (NVIC)** to pass data across a clean, multi-stage pipeline:

### 1. The Pendulum Reversal Point (Port K ISR)
When the moving pendulum triggers the limit marker, **`PK0`** fires an interrupt:
* The 8-bit LED ribbon (**Port M**) is instantly wiped to clear the previous sweep's display.
* A high signal is pulsed out of **`PD0`** for a strict hardware delay to kick off the sonic transducer burst.

### 2. The Flight-Time Stopwatch (Port D ISR)
The ultrasonic module pulls its `Echo` pin high, triggering **`PD1`**:
* **Rising Edge Detection:** The microcontroller instantly activates **Timer 1** in a 32-bit countdown configuration, operating like a high-speed microsecond stopwatch.
* **Falling Edge Detection:** The instant the bouncing sound wave returns, the stopwatch is frozen. The elapsed clock cycles are captured and translated directly into centimeters using an optimized divider pipeline factor:
  $$\text{Distance (cm)} = \frac{\text{Elapsed Cycles}}{933}$$

### 3. Progressive Visual Rendering
During the alternative swing phase, the calculated metric (`g_distance_cm`) is evaluated and output to the LED array:

* 📏 **< 10 cm:** `0x01` (1 LED on)
* 📏 **< 20 cm:** `0x03` (2 LEDs on)
* 📏 **< 30 cm:** `0x0F` (4 LEDs on)
* 📏 **< 40 cm:** `0x7F` (7 LEDs on)
* 🌌 **Out of Bounds / Far:** `0xFF` (All 8 LEDs on)

---

## ⚙️ Compilation, Flashing, & Deployment

### Toolchain Requirements
* **IDE:** Code Composer Studio (CCS) v12 or higher.
* **SDK Dependency:** TivaWare for C Series (specifically utilizing the direct register mappings found in `inc/tm4c1294ncpdt.h`).
