# Fastest Finger First Circuit

## Overview
This project implements a Fastest Finger First system designed for quiz competitions or any event where reaction speed matters. Each participant has a dedicated button, and the system instantly detects the first person to press theirs. A seven segment display shows a countdown before accepting inputs, ensuring a fair start.

## Features
Instant detection of the first button press  
Countdown display using a seven segment module  
Automatic lockout after the first valid input  
Manual reset to prepare for the next round  
Simple wiring for easy assembly

## Components Used
Microcontroller board (ESP32 based)  
Seven segment display  
Push buttons (one per participant)  
Resistors  
Jumper wires  
Breadboard or PCB

## Circuit Description
The microcontroller constantly monitors all participant buttons. When the start command is given, the seven segment display runs a countdown (for example 3-2-1). Once the countdown reaches zero, all buttons become active. The first button pressed is registered, and the system ignores further presses until reset. An LED or display can indicate the winner.

## Code

// Fastest Finger First — ESP32 + TM1637 4-digit 7-seg
// - Buttons use INPUT_PULLUP (wire each to GND)
// - TM1637 library: "TM1637Display" by Avishay Orpaz

#include <TM1637Display.h>

const uint8_t PIN_TM_CLK = 14;
const uint8_t PIN_TM_DIO = 27;

// Optional buzzer (active HIGH). Set to -1 to disable.
const int PIN_BUZZER = 26;

// Reset button (to start next round or clear result)
const uint8_t PIN_RESET = 25;

// Player buttons (use GPIOs that support digitalRead)
const uint8_t PLAYER_PINS[] = { 32, 33, 34, 35 };
const uint8_t NUM_PLAYERS = sizeof(PLAYER_PINS) / sizeof(PLAYER_PINS[0]);

// Countdown seconds before opening the buzzer window
const uint8_t COUNTDOWN_SECONDS = 3;

// Debounce (ms)
const uint32_t DEBOUNCE_MS = 25;

// Display brightness (0..7)
const uint8_t DISPLAY_BRIGHTNESS = 7;

TM1637Display display(PIN_TM_CLK, PIN_TM_DIO);

enum State { IDLE, COUNTDOWN, OPEN, LOCKED };
State state = IDLE;

int winner = -1;
uint32_t tOpen = 0;      // when window opened
uint32_t tPress = 0;     // when winner pressed

// Simple beep
void beep(uint16_t ms = 60) {
  if (PIN_BUZZER < 0) return;
  digitalWrite(PIN_BUZZER, HIGH);
  delay(ms);
  digitalWrite(PIN_BUZZER, LOW);
}

// Show "----"
void showDashes() {
  const uint8_t dash = SEG_G;
  uint8_t data[] = { dash, dash, dash, dash };
  display.setSegments(data);
}

// Show "rdy."
void showReady() {
  display.showString("rdy.");
}

// Show winner+time: [P][t2][t1][t0] e.g., P2 137ms -> "2137"
void showWinnerAndTime(int playerIdx, uint16_t ms) {
  if (ms > 999) ms = 999;
  uint8_t d[4];
  d[0] = display.encodeDigit((playerIdx + 1) % 10);
  d[1] = display.encodeDigit((ms / 100) % 10);
  d[2] = display.encodeDigit((ms / 10) % 10);
  d[3] = display.encodeDigit(ms % 10);
  display.setSegments(d);
}

// Basic debounced, active-LOW read
bool pressed(uint8_t pin) {
  static uint32_t lastChange[40] = {0};
  static uint8_t  lastStable[40];
  static bool initDone = false;

  if (!initDone) {
    initDone = true;
    for (int i = 0; i < 40; ++i) lastStable[i] = HIGH;
  }

  uint8_t idx = pin; // using GPIO number as index
  uint8_t r = digitalRead(pin);

  if (r != lastStable[idx]) {
    lastChange[idx] = millis();
    lastStable[idx] = r;
  }

  if (millis() - lastChange[idx] > DEBOUNCE_MS) {
    return (r == LOW); // active LOW
  }
  return false;
}

void startCountdown() {
  state = COUNTDOWN;
  for (int s = COUNTDOWN_SECONDS; s >= 1; --s) {
    // Rightmost digit shows 3,2,1
    display.showNumberDecEx(s, 0, true, 1, 3);
    beep(50);
    delay(750);
    showDashes();
    delay(150);
  }
  state = OPEN;
  tOpen = millis();
  showReady();
  beep(150);
}

void resetRound() {
  winner = -1;
  tPress = 0;
  state = IDLE;
  showDashes();
}

void setup() {
  display.setBrightness(DISPLAY_BRIGHTNESS, true);

  for (uint8_t i = 0; i < NUM_PLAYERS; ++i) {
    pinMode(PLAYER_PINS[i], INPUT_PULLUP);
  }
  pinMode(PIN_RESET, INPUT_PULLUP);

  if (PIN_BUZZER >= 0) {
    pinMode(PIN_BUZZER, OUTPUT);
    digitalWrite(PIN_BUZZER, LOW);
  }

  showDashes();
}

void loop() {
  // RESET button:
  //   - If IDLE, start a new round (countdown)
  //   - If LOCKED, clear and go back to IDLE
  if (pressed(PIN_RESET)) {
    if (state == IDLE) {
      startCountdown();
      delay(200); // avoid double trigger
    } else if (state == LOCKED) {
      resetRound();
      delay(200);
    }
  }

  // OPEN window: first valid player press wins
  if (state == OPEN) {
    for (uint8_t i = 0; i < NUM_PLAYERS; ++i) {
      if (pressed(PLAYER_PINS[i])) {
        winner = i;
        tPress = millis();
        uint16_t reaction = (uint16_t)(tPress - tOpen);
        showWinnerAndTime(winner, reaction);
        beep(200);
        state = LOCKED;
        break;
      }
    }
  }
}

## Code Functionality
Initialization – Sets up the seven segment display, button pins, and variables  
Countdown – Runs a non-blocking timer to display the countdown without freezing the program  
Input Detection – Continuously checks all buttons after the countdown  
Winner Lock – Records the first valid press and locks out others  
Reset Mechanism – Clears the winner and prepares the system for the next round

## How to Use
1. Build the circuit according to the schematic  
2. Upload the provided code to the microcontroller via Arduino IDE  
3. Power up the circuit  
4. Wait for the countdown to finish  
5. Press your button — if you are the fastest, you win  
6. Press the reset button to start again


[project PDF guide](./FastestFingerFirst.pdf).
