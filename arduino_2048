#include <Arduino.h>
#include <U8g2lib.h>
#include <avr/io.h>
#include <avr/interrupt.h>

// LCD ST7920 128x64
U8G2_ST7920_128X64_F_SW_SPI u8g2(
  U8G2_R0,
  13,  // E / SCLK
  11,  // R/W / SID
  10,  // RS / CS
  8    // RST
);



// D2-D7 sunt pe PORTD
#define BTN_UP_BIT     PD2
#define BTN_DOWN_BIT   PD3
#define BTN_LEFT_BIT   PD4
#define BTN_RIGHT_BIT  PD5
#define BUZZER_BIT     PD6
#define LED_R_BIT      PD7

// D9 si D12 sunt pe PORTB
#define LED_G_BIT      PB1   // Arduino D9
#define LED_B_BIT      PB4   // Arduino D12

int grid[4][4];
long score = 0;
bool gameOver = false;
bool won = false;

volatile bool buzzerEnabled = false;
volatile unsigned int buzzerTicks = 0;


void gpioInitRegisters() {
  // Butoane D2-D5 ca input
  DDRD &= ~((1 << BTN_UP_BIT) |
            (1 << BTN_DOWN_BIT) |
            (1 << BTN_LEFT_BIT) |
            (1 << BTN_RIGHT_BIT));

  // pull-up
  PORTD |= (1 << BTN_UP_BIT) |
           (1 << BTN_DOWN_BIT) |
           (1 << BTN_LEFT_BIT) |
           (1 << BTN_RIGHT_BIT);

  // Buzzer D6 ca output
  DDRD |= (1 << BUZZER_BIT);

  // LED Rosu D7 ca output
  DDRD |= (1 << LED_R_BIT);

  // LED Verde D9 si Albastru D12 ca output
  DDRB |= (1 << LED_G_BIT) | (1 << LED_B_BIT);

  PORTD &= ~((1 << BUZZER_BIT) | (1 << LED_R_BIT));
  PORTB &= ~((1 << LED_G_BIT) | (1 << LED_B_BIT));
}

bool buttonUpPressed() {
  return !(PIND & (1 << BTN_UP_BIT));
}

bool buttonDownPressed() {
  return !(PIND & (1 << BTN_DOWN_BIT));
}

bool buttonLeftPressed() {
  return !(PIND & (1 << BTN_LEFT_BIT));
}

bool buttonRightPressed() {
  return !(PIND & (1 << BTN_RIGHT_BIT));
}

void setLedRegisters(bool r, bool g, bool b) {
  if(r)
    PORTD |= (1 << LED_R_BIT);
  else
    PORTD &= ~(1 << LED_R_BIT);

  if(g)
    PORTB |= (1 << LED_G_BIT);
  else
    PORTB &= ~(1 << LED_G_BIT);

  if(b)
    PORTB |= (1 << LED_B_BIT);
  else
    PORTB &= ~(1 << LED_B_BIT);
}

void uartInitRegisters(unsigned long baud) {
  unsigned int ubrr = 16000000UL / 16 / baud - 1;

  UBRR0H = (unsigned char)(ubrr >> 8);
  UBRR0L = (unsigned char)ubrr;

  // Activare transmitter UART
  UCSR0B = (1 << TXEN0);

  // 8 biti date, 1 bit stop
  UCSR0C = (1 << UCSZ01) | (1 << UCSZ00);
}

void uartSendChar(char c) {
  while (!(UCSR0A & (1 << UDRE0))) {
    // asteapta pana cand bufferul este liber
  }
  UDR0 = c;
}

void uartPrint(const char *s) {
  while (*s) {
    uartSendChar(*s++);
  }
}

void uartPrintln(const char *s) {
  uartPrint(s);
  uartSendChar('\r');
  uartSendChar('\n');
}

void uartPrintNumber(long n) {
  char buf[16];
  ltoa(n, buf, 10);
  uartPrint(buf);
}

void timer2InitForBuzzer() {
  cli();

  TCCR2A = (1 << WGM21);
  TCCR2B = 0;

  OCR2A = 124;

  TIMSK2 = (1 << OCIE2A);

  TCCR2B = (1 << CS22);

  sei();
}

ISR(TIMER2_COMPA_vect) {
  if (buzzerEnabled && buzzerTicks > 0) {
    PORTD ^= (1 << BUZZER_BIT);
    buzzerTicks--;
  } else {
    buzzerEnabled = false;
    PORTD &= ~(1 << BUZZER_BIT);
  }
}

void beepRegisters(unsigned int durationMs) {
  // Timerul intrerupe la 0.5 ms, deci 1 ms = 2 ticks
  buzzerTicks = durationMs * 2;
  buzzerEnabled = true;
}

void clearGrid() {
  for (int i = 0; i < 4; i++) {
    for (int j = 0; j < 4; j++) {
      grid[i][j] = 0;
    }
  }
}

void addRandomTile() {
  int emptyCount = 0;

  for (int i = 0; i < 4; i++) {
    for (int j = 0; j < 4; j++) {
      if (grid[i][j] == 0) {
        emptyCount++;
      }
    }
  }

  if (emptyCount == 0) return;

  int pos = random(emptyCount);
  int count = 0;

  for (int i = 0; i < 4; i++) {
    for (int j = 0; j < 4; j++) {
      if (grid[i][j] == 0) {
        if (count == pos) {
          grid[i][j] = (random(10) == 0) ? 4 : 2;
          return;
        }
        count++;
      }
    }
  }
}

void initGame() {
  clearGrid();

  score = 0;
  gameOver = false;
  won = false;

  addRandomTile();
  addRandomTile();

  setLedRegisters(false, false, true); // albastru la start
  uartPrintln("Joc nou");
}

void drawGame() {
  u8g2.clearBuffer();

  int startX = 0;
  int startY = 0;
  int cell = 16;

  // Tabla 4x4
  for (int i = 0; i <= 4; i++) {
    u8g2.drawLine(startX, startY + i * cell, startX + 64, startY + i * cell);
    u8g2.drawLine(startX + i * cell, startY, startX + i * cell, startY + 64);
  }

  // Valorile din celule
  u8g2.setFont(u8g2_font_5x8_tf);
  char buf[8];

  for (int i = 0; i < 4; i++) {
    for (int j = 0; j < 4; j++) {
      if (grid[i][j] != 0) {
        sprintf(buf, "%d", grid[i][j]);

        int x = startX + j * cell + 2;
        int y = startY + i * cell + 11;

        u8g2.drawStr(x, y, buf);
      }
    }
  }

  // Zona din dreapta
  u8g2.setFont(u8g2_font_6x10_tf);
  u8g2.drawStr(72, 10, "2048");

  u8g2.setFont(u8g2_font_5x8_tf);
  u8g2.drawStr(70, 24, "Score:");

  char scoreBuf[14];
  ltoa(score, scoreBuf, 10);
  u8g2.drawStr(70, 34, scoreBuf);

  if (won) {
    u8g2.drawStr(70, 48, "WIN!");
  }

  if (gameOver) {
    u8g2.drawStr(70, 48, "GAME");
    u8g2.drawStr(70, 58, "OVER");
  }

  u8g2.sendBuffer();
}

bool compressLine(int line[4]) {
  bool moved = false;
  int temp[4] = {0, 0, 0, 0};
  int index = 0;

  for (int i = 0; i < 4; i++) {
    if (line[i] != 0) {
      temp[index] = line[i];
      index++;
    }
  }

  for (int i = 0; i < 4; i++) {
    if (line[i] != temp[i]) {
      moved = true;
    }
    line[i] = temp[i];
  }

  return moved;
}

bool mergeLine(int line[4]) {
  bool moved = false;

  for (int i = 0; i < 3; i++) {
    if (line[i] != 0 && line[i] == line[i + 1]) {
      line[i] *= 2;
      score += line[i];
      line[i + 1] = 0;
      moved = true;

      if (line[i] == 2048) {
        won = true;
      }
    }
  }

  return moved;
}

bool moveLeft() {
  bool moved = false;

  for (int i = 0; i < 4; i++) {
    int line[4];

    for (int j = 0; j < 4; j++) {
      line[j] = grid[i][j];
    }

    bool a = compressLine(line);
    bool b = mergeLine(line);
    bool c = compressLine(line);

    for (int j = 0; j < 4; j++) {
      grid[i][j] = line[j];
    }

    if (a || b || c) {
      moved = true;
    }
  }

  return moved;
}

bool moveRight() {
  bool moved = false;

  for (int i = 0; i < 4; i++) {
    int line[4];

    for (int j = 0; j < 4; j++) {
      line[j] = grid[i][3 - j];
    }

    bool a = compressLine(line);
    bool b = mergeLine(line);
    bool c = compressLine(line);

    for (int j = 0; j < 4; j++) {
      grid[i][3 - j] = line[j];
    }

    if (a || b || c) {
      moved = true;
    }
  }

  return moved;
}

bool moveUp() {
  bool moved = false;

  for (int j = 0; j < 4; j++) {
    int line[4];

    for (int i = 0; i < 4; i++) {
      line[i] = grid[i][j];
    }

    bool a = compressLine(line);
    bool b = mergeLine(line);
    bool c = compressLine(line);

    for (int i = 0; i < 4; i++) {
      grid[i][j] = line[i];
    }

    if (a || b || c) {
      moved = true;
    }
  }

  return moved;
}

bool moveDown() {
  bool moved = false;

  for (int j = 0; j < 4; j++) {
    int line[4];

    for (int i = 0; i < 4; i++) {
      line[i] = grid[3 - i][j];
    }

    bool a = compressLine(line);
    bool b = mergeLine(line);
    bool c = compressLine(line);

    for (int i = 0; i < 4; i++) {
      grid[3 - i][j] = line[i];
    }

    if (a || b || c) {
      moved = true;
    }
  }

  return moved;
}

bool isGameOver() {
  // Spatiu liber jocul continua
  for (int i = 0; i < 4; i++) {
    for (int j = 0; j < 4; j++) {
      if (grid[i][j] == 0) {
        return false;
      }
    }
  }

  // Jocul plin dar am pe linie 2 valori egale sau 2 pe coloana
  for (int i = 0; i < 4; i++) {
    for (int j = 0; j < 3; j++) {
      if (grid[i][j] == grid[i][j + 1]) {
        return false;
      }
    }
  }

  for (int j = 0; j < 4; j++) {
    for (int i = 0; i < 3; i++) {
      if (grid[i][j] == grid[i + 1][j]) {
        return false;
      }
    }
  }

  return true;
}

void processMove(bool moved) {
  if (moved) {
    addRandomTile();

    beepRegisters(70);
    setLedRegisters(false, true, false); // verde

    uartPrint("Mutare valida. Scor=");
    uartPrintNumber(score);
    uartPrintln("");
  } else {
    beepRegisters(180);
    setLedRegisters(true, false, false); // rosu

    uartPrintln("Mutare invalida");
  }

  if (isGameOver()) {
    gameOver = true;
    setLedRegisters(true, false, false);
    beepRegisters(500);

    uartPrintln("Game over");
  }

  if (won) {
    setLedRegisters(false, true, false);
    uartPrintln("Ai ajuns la 2048");
  }

  drawGame();
  delay(180);
}

void setup() {
  gpioInitRegisters();
  uartInitRegisters(9600);
  timer2InitForBuzzer();

  randomSeed(analogRead(A0));

  u8g2.begin();

  uartPrintln("Pornire proiect 2048");
  initGame();
  drawGame();
}

void loop() {
  if (gameOver) {
    if (buttonUpPressed() ||
        buttonDownPressed() ||
        buttonLeftPressed() ||
        buttonRightPressed()) {
      initGame();
      drawGame();
      delay(300);
    }

    return;
  }

  if (buttonUpPressed()) {
    uartPrintln("Buton SUS");
    processMove(moveUp());
  }

  if (buttonDownPressed()) {
    uartPrintln("Buton JOS");
    processMove(moveDown());
  }

  if (buttonLeftPressed()) {
    uartPrintln("Buton STANGA");
    processMove(moveLeft());
  }

  if (buttonRightPressed()) {
    uartPrintln("Buton DREAPTA");
    processMove(moveRight());
  }
}


