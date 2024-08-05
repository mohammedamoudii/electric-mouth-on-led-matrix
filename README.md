# electric-mouth-on-led-matrix


```
// This version uses bit-banged SPI.
// If you see tearing (jagged edges on the circles) try the version
// which uses AVR's hardware SPI peripheral:
// https://wokwi.com/arduino/projects/318868939929027156

#define CLK 13
#define DIN 11
#define CS  10
#define X_SEGMENTS   4
#define Y_SEGMENTS   4
#define NUM_SEGMENTS (X_SEGMENTS * Y_SEGMENTS)

// A framebuffer to hold the state of the entire matrix of LEDs
// laid out in raster order, with (0, 0) at the top-left
byte fb[8 * NUM_SEGMENTS];

void shiftAll(byte send_to_address, byte send_this_data)
{
  digitalWrite(CS, LOW);
  for (int i = 0; i < NUM_SEGMENTS; i++) {
    shiftOut(DIN, CLK, MSBFIRST, send_to_address);
    shiftOut(DIN, CLK, MSBFIRST, send_this_data);
  }
  digitalWrite(CS, HIGH);
}

void setup() {
  Serial.begin(115200);
  pinMode(CLK, OUTPUT);
  pinMode(DIN, OUTPUT);
  pinMode(CS, OUTPUT);

  // Setup each MAX7219
  shiftAll(0x0f, 0x00); // display test register - test mode off
  shiftAll(0x0b, 0x07); // scan limit register - display digits 0 thru 7
  shiftAll(0x0c, 0x01); // shutdown register - normal operation
  shiftAll(0x0a, 0x0f); // intensity register - max brightness
  shiftAll(0x09, 0x00); // decode mode register - No decode
  clear();
}

void loop() {
  clear();
  drawMouth();
  show();
  delay(1000);
  clear();
  show();
  delay(1000);
}

void drawMouth() {
  // Coordinates for drawing a simple mouth shape
  for (int x = 8; x < 24; x++) {
    safe_pixel(x, 16, 1); // upper lip
    safe_pixel(x, 18, 1); // lower lip
  }
  for (int y = 16; y < 19; y++) {
    safe_pixel(8, y, 1); // left corner of the mouth
    safe_pixel(23, y, 1); // right corner of the mouth
  }
}

void set_pixel(uint8_t x, uint8_t y, uint8_t mode) {
  byte *addr = &fb[x / 8 + y * X_SEGMENTS];
  byte mask = 128 >> (x % 8);
  switch (mode) {
    case 0: // clear pixel
      *addr &= ~mask;
      break;
    case 1: // plot pixel
      *addr |= mask;
      break;
    case 2: // XOR pixel
      *addr ^= mask;
      break;
  }
}

void safe_pixel(uint8_t x, uint8_t y, uint8_t mode) {
  if ((x >= X_SEGMENTS * 8) || (y >= Y_SEGMENTS * 8))
    return;
  set_pixel(x, y, mode);
}

// Turn off every LED in the framebuffer
void clear() {
  byte *addr = fb;
  for (byte i = 0; i < 8 * NUM_SEGMENTS; i++)
    *addr++ = 0;
}

// Send the raster order framebuffer in the correct order
// for the boustrophedon layout of daisy-chained MAX7219s
void show() {
  for (byte row = 0; row < 8; row++) {
    digitalWrite(CS, LOW);
    byte segment = NUM_SEGMENTS;
    while (segment--) {
      byte x = segment % X_SEGMENTS;
      byte y = segment / X_SEGMENTS * 8;
      byte addr = (row + y) * X_SEGMENTS;

      if (segment & X_SEGMENTS) { // odd rows of segments
        shiftOut(DIN, CLK, MSBFIRST, 8 - row);
        shiftOut(DIN, CLK, LSBFIRST, fb[addr + x]);
      } else { // even rows of segments
        shiftOut(DIN, CLK, MSBFIRST, 1 + row);
        shiftOut(DIN, CLK, MSBFIRST, fb[addr - x + X_SEGMENTS - 1]);
      }
    }
    digitalWrite(CS, HIGH);
  }
}
```

## https://wokwi.com/projects/404580755235719169
