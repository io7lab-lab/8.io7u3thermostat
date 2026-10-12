io7-3device-thermostat
======================

Themostat with 3 devices of valve, thermometer, and the rotary controller
<img width="911" height="311" alt="Screenshot 2025-09-17 at 12 40 24 PM" src="https://github.com/user-attachments/assets/b00ef70d-8391-481a-953f-27689276bd3e" />

### io7uRotary
```python
from IO7FuPython import ConfiguredDevice
import json
import time
import uComMgr32
from machine import Pin
import st7789
import tft_config
import vga2_8x16 as font1
import vga1_bold_16x32 as font2

rotaryA = Pin(43, Pin.IN, Pin.PULL_UP)
rotaryB = Pin(44, Pin.IN, Pin.PULL_UP)
switch  = Pin(18, Pin.IN, Pin.PULL_UP)

old_encode = 0
right = [13, 32, 20, 01]
left = [10, 02, 23, 31]

switch_state = 'none'
def pushed(p):
    global lastPub, switch_state
    switch_state = 'clicked'
    lastPub = time.ticks_ms() - device.meta['pubInterval'] + 30
        
value = 0
def rotated(p):
    global old_encode, value, lastPub
    new_state = rotaryA.value() << 1| rotaryB.value()
    if new_state != old_encode:
        new_combined = old_encode * 10 + new_state
        old_encode = new_state
        if new_combined in right:
            value = (value + 1) if value < 255 else 255
        elif new_combined in left:
            value = (value - 1) if value > 0 else 0
        lastPub = time.ticks_ms() - device.meta['pubInterval'] + 50
        display()

rotaryA.irq(trigger=Pin.IRQ_FALLING | Pin.IRQ_RISING, handler=rotated)
rotaryB.irq(trigger=Pin.IRQ_FALLING | Pin.IRQ_RISING, handler=rotated)
switch.irq(trigger=Pin.IRQ_FALLING, handler=pushed)

tft = tft_config.config(3, buffer_size=64*64*2)

tft.init()
tft.fill(st7789.BLACK)
tft.text(font2, 'Dimmer Encoder', 45, 5, st7789.WHITE)

tft.text(font1, 'Level : ', 65, 80, st7789.WHITE)

def handleCommand(topic, msg):
    global lastPub, value
    jo = json.loads(str(msg,'utf8'))
    if ("value" in jo['d']):
        value = int(jo['d']['value'])
        lastPub = - device.meta['pubInterval']
        display()

def display():
    tft.text(font1, f'{value:3}', 200, 80, st7789.WHITE)
        
nic = uComMgr32.startWiFi('rotary')
device = ConfiguredDevice()
device.setUserCommand(handleCommand)

device.connect()

lastPub = time.ticks_ms() - device.meta['pubInterval']

display()

while True:
    if not device.loop():
        tft.deinit()
        break
    if (time.ticks_ms() - device.meta['pubInterval']) > lastPub:
        lastPub = time.ticks_ms()
        device.publishEvent('status',
            json.dumps({
                'd' : {
                        'value': value,
                        'switch' : switch_state
                      }
                 }
            )
        )
        switch_state = 'none'

```

### io7uThermometer
```python
from IO7FuPython import ConfiguredDevice
import json
import time
import uComMgr32
import dht
from machine import Pin


def handleCommand(topic, msg):
    pass

nic = uComMgr32.startWiFi('iot')
device = ConfiguredDevice()
device.setUserCommand(handleCommand)

device.connect()
sensor = dht.DHT22(Pin(16))

lastPub = time.ticks_ms() - device.meta['pubInterval']

while True:
    if not device.loop():
        break
    if (time.ticks_ms() - device.meta['pubInterval']) > lastPub:
        lastPub = time.ticks_ms()
        sensor.measure()
        device.publishEvent('status', json.dumps({'d':{'temperature': sensor.temperature(),
                                                       'humidity': sensor.humidity()
                                                       }
                                                  }))
```

### io7uValve
```python
from IO7FuPython import ConfiguredDevice
import json
import time
import uComMgr32
from machine import Pin

def handleCommand(topic, msg):
    global lastPub
    jo = json.loads(str(msg,'utf8'))
    if ("valve" in jo['d']):
        if jo['d']['valve'] == 'on':
            led.on()
        else:
            led.off()
        lastPub = - device.meta['pubInterval']

nic = uComMgr32.startWiFi('iot')
device = ConfiguredDevice()
device.setUserCommand(handleCommand)

device.connect()

led = Pin(13, Pin.OUT)
lastPub = time.ticks_ms() - device.meta['pubInterval']

while True:
    if not device.loop():
        break
    if (time.ticks_ms() - device.meta['pubInterval']) > lastPub:
        lastPub = time.ticks_ms()
        device.publishEvent('status', json.dumps({'d':{'valve': 'on' if led.value() else 'off'}}))
```
