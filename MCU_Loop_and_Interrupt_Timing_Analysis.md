# Performance and interrupt latency of popular IoT boards and MCUs

An easy way to check out well-known IoT boards and MCUs in terms of their "baseline" performance and their interrupt latency. 

There are two test methods that switch the GPIOs where the measurements are made: 
1. minimum **loop()** without interrupt
2. minimum **loop()** with triggering of interrupt service routine **isr()**. 

The simplest way to make a measurement is to use a 4 or 8 channel logic analyser or alternatively a dual channel digital scope

```text
+--------+                                                    +----------------+
|        | ------ PIN_PORT_ISR* ----------- ch. 5 (green) --- | LOGIC ANALYSER |
|  DUT   | ------ PIN_PORT_INT -- -----+----ch. 6 (blue)  --- | OR DUAL SCOPE  <
|        | ------ PIN_PORT_ALT_INT ----|                      +----------------+
+--------+

*) RUN_LOOP_INTERRUPTED YES

Fig. 1 - Device under test (DUT) and logic analyser or dual channel digital scope
```

Normally two GPIOs are sufficient (PIN_PORT_ISR and PIN_PORT_INT). PIN_PORT_INT indicates loop activity as a common GPIO port and simultaneously triggers the associated interrupt service routine isr() in interrupt test mode (RUN_LOOP_INTERRUPTED YES) on this port PIN_PORT_INT. PIN_PORT_ISR shows the response from isr(). 

There are breakout boards where a separate interrupt input port (PIN_PORT_ALT_INT) must be configured and connected externally to PIN_PORT_INT.

Disclaimer: This is not a scientific and systematic comprehensive study, but only a **quick, cheap method** to better assess the different MCUs in terms of their basic properties and limitations. Word-width influences or multicore properties, for example, are not examined. 

## Draft Arduino sketch v0.1.6

It is important to note that there is no assembler optimisation, only native Arduino/C++ code is used.

```cpp
/*
    MCU Loop and Interrupt Timing Analysis

    Setting up the measurement:      
      8 channel logic analyser (Saleae original or clone) is recommended.
      Software used here is Saleae v2.4.7
      PIN_PORT_INT Connection to the logic analyser channel 6 (blue) is recommended.
      PIN_PORT_ISR Connection to the logic analyser channel 5 (green) is recommended.
      In some cases, if the interrupt handler requires a dedicated input pin,
      PIN_PORT_ALT_INT Connection to the logic analyser channel 7 (purple) is recommended.

    Version:    0.1.6
    Author:     artkeller@gmx.de
    Copyright:  2023 Thomas Walloschke
    Date:       2023-05-09
    Update:     2023-06-10
    Licences:   CC BY 4.0 and see Arduino.h (e.g. GNU Lesser General Public License)

*/

/* Check environment */
#ifdef ARDUINO        // run in Arduino ecosystem only
#include <Arduino.h>  // selection depends on MCU
#define YES 1
#define NO 0

/* Select interrupt test mode */
#define RUN_LOOP_INTERRUPTED YES

/* MCU identification */
// -- ATMEGA 85
#if defined(ARDUINO_AVR_TRINKET3)  /* ATTN: DRIVER NOT SUPPORTED SINCE 2012 */ \
  || defined(ARDUINO_AVR_TRINKET5) /* ATTN: DRIVER NOT SUPPORTED SINCE 2012 */
#warning "ATTN: ARDUINO_AVR_TRINKET"
#define PIN_PORT_INT 2  // TRIGGER & INT  (blue)  SHARED PIN
#define PIN_PORT_ISR 1  //                (green)
#endif                  // ARDUINO_AVR_TRINKET*

#if defined(ARDUINO_AVR_PROTRINKET3)  /* ATTN: DRIVER NOT SUPPORTED SINCE 2012 */ \
  || defined(ARDUINO_AVR_PROTRINKET5) /* ATTN: DRIVER NOT SUPPORTED SINCE 2012 */
#warning "ATTN: ARDUINO_AVR_PROTRINKET"
#define PIN_PORT_INT 2  // TRIGGER & INT  (blue)  SHARED PIN
#define PIN_PORT_ISR 5  //                (green)
#endif                  // ARDUINO_AVR_PROTRINKET*

// -- AVR Family
#if defined(ARDUINO_AVR_UNO) \
  || defined(ARDUINO_AVR_MEGA2560) \
  || defined(ARDUINO_ARCH_AVR)
#warning "ARDUINO_ARCH_AVR"
#define PIN_PORT_INT 2  // TRIGGER & INT  (blue)  SHARED PIN
#define PIN_PORT_ISR 5  //                (green)
#endif                  // ARDUINO_AVR_*

// -- SAMD21 Family
#ifdef ARDUINO_TRINKET_M0
#warning "ARDUINO_TRINKET_M0"
#define PIN_PORT_ALT_INT 0  // dedicated INT  (purple) CONNECT TO PIN_PORT_INT
#define PIN_PORT_INT 2      // TRIGGER        (blue)   CONNECT TO PIN_PORT_ALT_INT
#define PIN_PORT_ISR 1      //                (green)
#endif                      // ARDUINO_TRINKET_M0

// -- ESP Family (single core measurment - INT on same core)
#if not defined(ARDUINO_ARCH_ESP8266) \
  && defined(ESP_PLATFORM)
#warning "ESP_PLATFORM"
#define PIN_PORT_INT 2  // TRIGGER & INT  (blue)  SHARED PIN
#define PIN_PORT_ISR 5  //                (green)

#elif defined(ARDUINO_ARCH_ESP8266) \
  && defined(ARDUINO_ESP8266_WEMOS_D1R1)
#warning "ATTN: ARDUINO_ESP8266_WEMOS_D1R1"
#define PIN_PORT_INT D3  // TRIGGER & INT  (blue)  SHARED PIN - ATTN: don't use D2
#define PIN_PORT_ISR D5  //                (green)
#endif                   // ARDUINO_ESP8266_WEMOS_D1R1

// -- RPi Pico 2040 Family (single core measurment - INT on same core)
#ifdef TARGET_RP2040
#warning "TARGET_RP2040"
#define PIN_PORT_INT 21  // TRIGGER & INT  (blue)  SHARED PIN
#define PIN_PORT_ISR 20  //                (green)
#endif                   // TARGET_RP2040

#if not(defined(PIN_PORT_INT) \
        and defined(PIN_PORT_ISR))
#error "MCU UNSUPPORTED"
#endif

/* Set up the output port(s) for the logic analyser */
void setup() {
  digitalWrite(PIN_PORT_INT, false);  // prevent glitch
  pinMode(PIN_PORT_INT, OUTPUT);      // activate interrupt trigger
  digitalWrite(PIN_PORT_ISR, false);  // prevent glitch
  pinMode(PIN_PORT_ISR, OUTPUT);      // activate interrupt service signal

#if RUN_LOOP_INTERRUPTED == YES
#warning "with interrupt handling"
#ifdef PIN_PORT_ALT_INT
#warning "The alternative interrupt port must have a hardware connection to the interrupt trigger"
  pinMode(PIN_PORT_ALT_INT, INPUT_PULLUP);  // alternative interrupt port
  attachInterrupt(digitalPinToInterrupt(PIN_PORT_ALT_INT), isr, RISING);
#else
  attachInterrupt(digitalPinToInterrupt(PIN_PORT_INT), isr, RISING);
#endif

  /* Enable all interrupts */
  void __enable_irq(void);  // may be redundant, but to be on the safe side
#else
#warning "without interrupt handling"

  /* Disable all interrupts */
  void __disable_irq(void);  // to be on the safe side
#endif
}

/* Interrupt handling */
#if RUN_LOOP_INTERRUPTED == YES
// Interrupt service routine for the measurement of the latency

#ifdef ARDUINO_ARCH_ESP8266
ICACHE_RAM_ATTR  // ESP8266 architecture interrupt handler requires ICACHE_RAM_ATTR
#endif
void isr() {
  digitalWrite(PIN_PORT_ISR, true);   // ISR start and interrupt response
  digitalWrite(PIN_PORT_ISR, false);  // ISR end
}
#endif // RUN_LOOP_INTERRUPTED

/* Continuous minimum loop for logic analysis */
void loop() {
  digitalWrite(PIN_PORT_INT, true);   // loop start and interrupt trigger
  digitalWrite(PIN_PORT_INT, false);  // loop end
}

#else
#error "SORRY: ENVIRONMENT UNSUPPORTED - ARDUINO ONLY"
#endif  // ARDUINO

```
## Timing diagram

The following diagram terms t0..t5 are used for the time series measurements:

<img width="576" alt="Timing" src="https://github.com/artkeller/tobedefined/assets/16447285/3492d8d6-adde-4665-a875-2025c735dd1c">

## Pattern

The measurements showed that there are not only continuous signals. In the "background", i.e. below the loop() routine used by the MCU, there are different time series patterns.

First, the undisturbed continuous pattern (A) is measured. Then the other patterns (B...) are analysed and also measured. The measured pattern is shown in the tables.

## Measurements

All measurements are based on compilations in June 2023 with the **Arduino IDE Version 2.1.0, 2023-04-19T15:31:10.185Z, CLI Version 0.32.2** and the latest libraries for each MCU architecture.

It was measured using an 8-channel logic analyser (Saleae clone) with Saleae software v2.4.7.

### RUN_LOOP_INTERRUPTED=NO

| Board                 | MCU        | CPU_Freq  | Ratio | Pat | t0 [us] | f0 [kHz] | Clk Cyc | t1 [us] | t2 [us] | Duty [%] |
| --------------------- | ---------- | --------: | ----: | :-: | ------: | -------: | ------: | ------: | ------: | -------: |
| Arduino Uno           | ATMEGA328P | 16000000  | 1.0   | A   | 6.750   |  148.148 | 108     | 3.167   | 3.583   |  46.91   |
| Arduino 2560          | ATMEGA2569 | 16000000  | 1.0   | A   | 11.500  |   86.957 | 184     | 5.563   | 5.938   |  48,37   |
| Trinket M0            | SAMD21     | 48000000  | 3.0   | A   | 3.667   |  272.727 | 176     | 1,500   | 2.167   |  40.91   |
| Wemos D1R1            | ESP8266EX  | 80000000  | 5.0   | A   | 10.900  |   91.743 | 872     | 2.000   | 8.900   |  18.35   |
| Raspberry Pi Pico W   | RP2040     | 133000000 | 8.3   | A   | 1.625   |  615.385 | 216     | 0.625   | 1.000   |  37.46   | 
| Raspberry Pi Pico W   | RP2040     | 240000000 | 15.0  | A   | 0.917   | 1091.00  | 220     | 0.333   | 0.583   |  36.36   |
| M5Stack-Core-ESP32    | ESP32      | 240000000 | 15.0  | A   | 0.792   | 1263.00  | 190     | 0.333   | 0.458   |  42.11   |



### RUN_LOOP_INTERRUPTED=YES

| Board                 | MCU        | CPU_Freq  | Ratio | Pat | t0 [us] | f0 [kHz] | Clk Cyc | t1 [us] | t2 [us] | Duty [%] | t3 [us] | t4 [us] | t5 [us] |
| --------------------- | ---------- | --------: | ----: | :-: | ------: | -------: | ------: | ------: | ------: | -------: | ------: | ------: | ------: |
| Arduino Uno           | ATMEGA328P | 16000000  | 1.0   | A   | 21.208  |   47,244 |         | 17.333  |  3.874  |   81.73  |  7.333  |  4.292  | 5.708   |
| Arduino 2560          | ATMEGA2569 | 16000000  | 1.0   | A   |         |          |         |         |         |          |         |         |         |
| Trinket M0            | SAMD21     | 48000000  | 3.0   | A   |         |          |         |         |         |          |         |         |         |
| Wemos D1R1            | ESP8266EX  | 80000000  | 5.0   | A   |         |          |         |         |         |          |         |         |         |
| Raspberry Pi Pico W   | RP2040     | 133000000 | 8.3   | A   |         |          |         |         |         |          |         |         |         |
| Raspberry Pi Pico W   | RP2040     | 240000000 | 15.0  | A   |         |          |         |         |         |          |         |         |         |
| M5Stack-Core-ESP32    | ESP32      | 240000000 | 15.0  | A   |  4.125  |  242.424 |         |  3.583  |  0.542  |   86.87  | 1.917   | 0.333   | 1.333   |


