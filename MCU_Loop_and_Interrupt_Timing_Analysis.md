# "Base" performance and interrupt latency of popular IoT boards and MCUs

An easy way to check out well-known IoT boards and MCUs in terms of their "baseline" performance and their interrupt latency.

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

<img width="576" alt="Timing" src="https://github.com/artkeller/tobedefined/assets/16447285/3492d8d6-adde-4665-a875-2025c735dd1c">

| MCU                | INT | t0     | t1     | t2     | t3     | t4      | t5      | 
| ------------------ | --- | ------ | ------ | ------ | ------ | ------- | --------| 
| M5Stack-Core-ESP32 | NO  | 4.0 us | 2.0 us | 2.0 us | -      | -       | -       |
| M5Stack-Core-ESP32 | YES | 5.5 us | 3.5 us | 2.0 us | 1.0 us | 1.5 us  | 1.0 us  |
| Wemos D1R1         | NO  | 4.0 us | 2.0 us | 2.0 us | -      | -       | -       |
| Wemos D1R1         | YES | 4.0 us | 2.0 us | 2.0 us | 1.0 us | 1.5 us  | 1.0 us  |
| Arduino Uno        | NO  | 4.0 us | 2.0 us | 2.0 us | -      | -       | -       |
| Arduino Uno        | YES | 4.0 us | 2.0 us | 2.0 us | 1.0 us | 1.5 us  | 1.0 us  |
| Trnjet M9          | NO  | 4.0 us | 2.0 us | 2.0 us |  -     | -       | -       |
| Trnjet M9          | YES | 4.0 us | 2.0 us | 2.0 us | 1.0 us | 1.5 us  | 1.0 us  |
| RP2040             | NO  | 4.0 us | 2.0 us | 2.0 us |  -     | -       | -       |
| RP2040             | YES | 4.0 us | 2.0 us | 2.0 us | 1.0 us | 1.5 us  | 1.0 us  |

