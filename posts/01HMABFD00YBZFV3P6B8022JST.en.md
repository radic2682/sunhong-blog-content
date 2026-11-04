---
id: "01HMABFD00YBZFV3P6B8022JST"
title_en: "Building an Arduino Auto Fire Extinguisher: Flame, Smoke, Heat Sensing + IoT"
summary_en: "KURUNG, an Arduino auto fire extinguisher entered in Sungkyunkwan University's 2021 Creative Capstone Design Competition. We built a low-cost IoT device that judges flame, smoke, and temperature sensors together to detect kitchen fires early, sprays extinguishing agent, and alerts a smartphone."
---
![KURUNG auto fire extinguisher - Sungkyunkwan University Creative Capstone Design Competition|large](/img/01HMABFD00YBZFV3P6B8022JST/d0d89baf45a4.webp)

This is **KURUNG**, an auto fire extinguisher project entered in Sungkyunkwan University's 2021 Creative Capstone Design Competition. It is an IoT-based safety device that detects kitchen fires early and puts them out automatically, built as a team project.

## Background

Kitchen fires make up a large share of household fires, yet most homes have no device that can detect them early and respond immediately. Commercial automatic kitchen fire extinguishers were expensive and cumbersome to install, so they had not spread to ordinary households.

Our team set out to design an auto fire extinguisher that anyone could install at low cost. The core requirements were:

- Early fire detection (flame + smoke + temperature combined)
- Automatic suppression (spraying extinguishing agent)
- User notification (smartphone alert)
- Low-cost implementation (affordable for an ordinary household)

## Key Features

### Fire Detection with Multiple Sensors
A single sensor is prone to false alarms, so we combined data from several sensors to decide whether there was a fire:
- **Flame sensor**: Confirms an actual flame by sensing the infrared band
- **Temperature sensor**: Detects a sudden rise in temperature
- **Smoke sensor**: A gas/smoke sensor such as the MQ-2

We sampled sensor data over a fixed time window, gave points whenever a sensor crossed its threshold, and declared a fire only when the total score passed a set level, which cut down false alarms.

### Automatic Extinguishing Mechanism
When a fire is detected, an actuator sprays the extinguishing agent. A solenoid valve opens the agent container, and the nozzle angle is controlled to spray toward the source of the fire.

### IoT Integration
A WiFi module sends detection events to a server, which immediately pushes an alert to the user's smartphone. Users can check on the kitchen even when they are away.

## Hardware

- **MCU**: Arduino / ESP32 family
- **Sensors**: Flame sensor, temperature sensor (DHT22 or thermistor), gas/smoke sensor (MQ-2)
- **Actuators**: Solenoid valve, servo motor (spray direction control)
- **Communication**: WiFi module (ESP8266/ESP32)
- **Power**: 9V adapter + battery backup

## Software and Algorithm

The control logic was implemented as a finite state machine (FSM):

1. **IDLE**: Periodically monitors sensor values
2. **WARNING**: Some thresholds exceeded - early alert to the user
3. **FIRE_DETECTED**: Fire confirmed - start spraying agent
4. **EXTINGUISHING**: Spraying, periodically rechecking whether the fire persists
5. **COOLDOWN**: Waits after spraying, watching for reignition

Each state transition is decided by a combination of sensor scores and time thresholds. Rather than simply "smoke detected → spray," we designed it to spray only after several pieces of evidence accumulate.

## System Diagram and Details

The blog of our team leader has a detailed write-up of the full system architecture, schematics, and code.

[Take a look](https://velog.io/@tae_uk/%EC%9E%90%EB%8F%99%EC%86%8C%ED%99%94%EC%9E%A5%EC%B9%98-%EC%BF%A0%EB%A6%89-...-2-2021-%EC%84%B1%EA%B7%A0%EA%B4%80%EB%8C%80%ED%95%99%EA%B5%90-%EC%B0%BD%EC%9D%98%EC%A0%81-%EC%A2%85%ED%95%A9%EC%84%A4%EA%B3%84-%EA%B2%BD%EC%A7%84%EB%8C%80%ED%9A%8C)

## Takeaways

- **Preventing false alarms is the key**: Single-sensor detection produces too many false alarms to be practical. Multiple sensors + a time window + threshold tuning are essential
- **Choosing the actuator is hard**: Finding a solenoid valve suited to spraying extinguishing agent was trickier than expected. Pressure, flow rate, and price all had to be considered
- **Limits of the test environment**: Tests with real fire were limited for safety reasons. We verified at a resolution of several seconds rather than sub-second precision
- **Experience in integrated design**: The biggest lesson was integrating sensors, controller, actuators, and communication into one product. Building just one subsystem well is not enough
- **Teamwork**: We split the work across hardware, firmware, server, and app. Interfaces have to be clearly defined early on to avoid conflicts later

This project was a great chance to experience the full pipeline of an IoT device (sensing, decision, actuation, communication). Unlike a simple circuit exercise, it aimed to build a product that solves a real social problem, which made it highly motivating.
