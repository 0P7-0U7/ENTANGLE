<picture>
  <source media="(prefers-color-scheme: dark)" srcset="images/LOGO_DARK.png">
  <source media="(prefers-color-scheme: light)" srcset="images/LOGO_LIGHT.png">
  <img alt="LEADER by OPT-OUT" src="images/LOGO_LIGHT.png" width="100%">
</picture>


# ENTANGLE by OPT-OUT
A lightweight ESP32 library for real-time hardware mirroring via ESP-NOW. It creates a low-latency "virtual wire" between two nodes, automatically syncing GPIO states and PWM values through a distributed bitmask. No network stack required—just seamless, symmetric state synchronization for instant peer-to-peer hardware entanglement

No network stack or router is required. It provides seamless, symmetric state synchronization for instant peer-to-peer hardware entanglement.

## Core Features

* **Zero-Configuration Broadcasting:** Flash the exact same code to multiple boards. By broadcasting to the universal MAC address, nodes instantly entangle without hardcoding MAC addresses.
* **Microsecond Latency:** Bypasses slow `digitalRead()` / `digitalWrite()` functions. ENTANGLE reads and writes directly to the ESP32 hardware port registers (`GPIO.in`, `GPIO.out_w1ts`), executing state changes in 1-2 CPU cycles.
* **Hardware Safety Masking:** Automatically protects critical internal ESP32 pins (like SPI Flash and UART) from accidental overwrites during high-speed register manipulation.
* **Wi-Fi Channel Separation:** Isolate multiple pairs of entangled devices in the same physical space by locking them to different RF channels, preventing cross-talk.
* **Analog/PWM Synchronization:** Maps ADC inputs to PWM outputs with a built-in customizable deadband filter to eliminate network flooding caused by electrical noise.
* **Watchdog Failsafe:** Built-in sequence IDs drop stale packets, while a continuous heartbeat ensures that if a node loses power or range, the receiving node automatically drops all outputs to a safe LOW state.

## Installation

1. Download this repository as a `.zip` file.
2. In the Arduino IDE, navigate to **Sketch > Include Library > Add .ZIP Library...**
3. Select the downloaded file.

## Quick Start (Zero-Config Broadcast)

Flash this exact sketch to two different ESP32 boards. Connect a button to GPIO 4 on Node A, and an LED to GPIO 4 on Node B. When you press the button on Node A, Node B's LED will light up instantly.

```cpp
#include <WiFi.h>
#include "ENTANGLE.h"

ENTANGLE mirror;

// Universal Broadcast MAC Address
uint8_t broadcastAddress[] = {0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF}; 

void setup() {
    Serial.begin(115200);

    // Initialize ENTANGLE and lock to Wi-Fi Channel 4
    if (!mirror.begin(4)) {
        Serial.println("Fatal Error: Could not start ENTANGLE.");
        while (true); 
    }

    // Pair using the broadcast address
    mirror.addPeer(broadcastAddress);

    // Define hardware pins
    pinMode(4, INPUT_PULLUP); // Button input
    pinMode(5, OUTPUT);       // LED output
}

void loop() {
    // Handles state-change detection, transmission, and the watchdog timer
    mirror.update();
}

```

## How It Works

### High-Speed Register Masking

Instead of iterating through pins, ENTANGLE reads the entire 32-bit GPIO input register in a single cycle. It uses a bitwise XOR (`^`) against the last known state to detect changes. To prevent crashing the microcontroller, a `SAFE_GPIO_MASK` is applied before any write operation, ensuring pins 1 (TX), 3 (RX), and 6-11 (Internal Flash) are never accidentally toggled.

### The "Party Line" Fix (Channel Separation)

When using the Broadcast MAC address (`FF:FF:FF:FF:FF:FF`), any ESP32 running this library will mirror any other. If you want to deploy multiple independent pairs of devices in the same building, pass a specific channel number to the `begin()` method (e.g., `mirror.begin(6)`). This separates the devices at the physical radio frequency (RF) level, keeping your CPU overhead at absolute zero for rejected packets.

### Heartbeat & Watchdog

Because ESP-NOW is inherently connectionless, physical outputs could get stuck HIGH if a device suddenly powers off. ENTANGLE automatically transmits a heartbeat packet every 250ms. If the receiver does not hear a valid packet within 1000ms, the Watchdog engages and forces all safe GPIOs and PWM channels to 0.

## API Reference

### Initialization & Pairing

* `bool begin(uint8_t channel = 1)`: Initializes ESP-NOW and locks the radio to the specified Wi-Fi channel (1-14).
* `bool addPeer(const uint8_t* mac_addr)`: Registers a peer device. Accepts specific MAC addresses or the broadcast address.
* `void update()`: The main engine. Must be called continuously inside `loop()`.

### Analog & PWM

* `void attachAnalogInput(uint8_t channel, uint8_t pin)`: Maps a local analog pin to a virtual transmission channel (0-3).
* `void attachPWMOutput(uint8_t channel, uint8_t pin)`: Maps a virtual reception channel (0-3) to a local PWM output pin.
* `void setPWMThreshold(uint8_t threshold)`: Sets the ADC noise deadband (default is 3). An analog value must change by this amount to trigger a transmission.

### Timers & Safety

* `void setHeartbeatInterval(uint32_t interval_ms)`: Adjusts how often keep-alive packets are sent (default 250ms).
* `void setTimeout(uint32_t timeout_ms)`: Adjusts the watchdog timeout before the failsafe triggers (default 1000ms).

---

