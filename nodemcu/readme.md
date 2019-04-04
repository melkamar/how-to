# NodeMCU

On Mac

- Reference: https://naucse.python.cz/lessons/intro/micropython/ (the PORT for Mojave is `/dev/tty.usbserial-14420` instead of whatever is said in the article)
- Connect to repl: `screen /dev/tty.usbserial-14420 115200` (! after connecting, press the Flash button)
- Run script: `ampy -p /dev/tty.usbserial-14420 run script.py`

## Run lights back and forth
```python
from machine import Pin
from neopixel import NeoPixel
from time import sleep

NUM_LEDS = 15
pin = Pin(2, Pin.OUT)
np = NeoPixel(pin, NUM_LEDS)

color_val = 0
color_val_delta = 5

g_val = 0
g_delta = 3

b_val = 0
b_delta = 2

direction = 1
i = 0
while True:
    prev_idx = (i - direction + NUM_LEDS) % NUM_LEDS
    np[prev_idx] = (0, 0, 0)

    np[i] = (color_val, g_val, b_val)
    np.write()
    sleep(0.05)

    if i == 0:
        direction = 1
    if i == NUM_LEDS - 1:
        direction = -1

    i += direction

    if color_val <= 0:
        color_val_delta = 5
    if color_val >= 255:
        color_val_delta = -5
    color_val += color_val_delta

    if g_val <= 0:
        g_delta = 3
    if g_val >= 255:
        g_delta = -3
    g_val += g_delta

    if b_val <= 0:
        b_delta = 2
    if b_val >= 255:
        b_delta = -2
    b_val += b_delta
```
