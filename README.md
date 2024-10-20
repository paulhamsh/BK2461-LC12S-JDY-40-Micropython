# BK2461-LC12S-JDY-40-Micropython
Micropython to control the LC12S and JDY-40 wireless communication boards based on the BK2461


```
LC12S datasheet     https://arduinolab.pw/wp-content/uploads/2019/05/H2-LCS12.pdf
                    http://user.cavenet.com/jgurley/lc12s/LC12S_datasheet.pdf

JDY-40 datasheet    https://w.electrodragon.com/w/images/0/05/EY-40_English_manual.pdf

BK2461 datasheet    https://www.mikrocontroller.net/attachment/381910/BT-WiFi-52rf5541.pdf
```



## LC12S configuration

Byte       |  Value       | Description
-----------|--------------|---------------
1          |  0x44, 0x5a  |    Header
3          |  0x22, 0x11  |    Device Id (cannot be altered on LC12S)
5          |  0x33, 0x44  |    Network address
1          |  0x00        |    
1          |  0x06        |    Power
1          |  0x00        |    
1          |  0x05        |    Baud rate
1          |  0x00        |    
1          |  0x60        |    Channel
1          |  0x00        |    
1          |  0x00        |    
1          |  0x00        |    
1          |  0x12        |    Number of bytes in messages (18)
1          |  0x00        |    
1          |  0x00        |    CRC

0xAA, 0x5A, 0x22, 0x44, 0x11, 0x33, 0x00, 0x06, 0x00, 0x05, 0x00, 0x60, 0x00, 0x00, 0x00, 0x12, 0x00, 0x2e


## One Pico Pi running two LC12S.   
One UART on GP0, GP1, the second on GP4, GP5.   

UART        |   1   |    2
------------|-------|-------- 
RX          |  0    |    4
TX          |  1    |    5
CS          |  2    |    6
SET         |  3    |    7

```
import board
import digitalio
import busio
from digitalio import DigitalInOut, Direction, Pull
import time
import binascii

uart = busio.UART(tx=board.GP0, rx=board.GP1, baudrate=9600, timeout = 0.1)
uart1 = busio.UART(tx=board.GP4, rx=board.GP5, baudrate=9600, timeout = 0.1)


def print_hex(val):
    print("".join(" x%02x" % i for i in val))

def lc12_trf(uart, cmd):
    buf = bytes(cmd)
    print_hex(buf)
    uart.write(buf)
    time.sleep(0.5)
    leng = uart.in_waiting
    if leng > 0:
        val = uart.read(leng)
        print_hex(val)
    print()

csw=DigitalInOut(board.GP2)
csw.direction = Direction.OUTPUT
setw = DigitalInOut(board.GP3)
setw.direction = Direction.OUTPUT

csw1=DigitalInOut(board.GP6)
csw1.direction = Direction.OUTPUT
setw1 = DigitalInOut(board.GP7)
setw1.direction = Direction.OUTPUT

setw.value = True
csw.value = True
setw1.value = True
csw1.value = True

time.sleep(1.0)

setw.value = False
csw.value = False
setw1.value = False
csw1.value = False

time.sleep(1.0)

config = [0xAA, 0x5A, 0x22, 0x44, 0x11, 0x33, 0x00, 0x06, 0x00, 0x05, 0x00, 0x60, 0x00, 0x00, 0x00, 0x12, 0x00, 0x2e]

temp = 0
for i in range(17):
    temp += config[i]
config[17] = temp % 256

lc12_trf(uart, config)
lc12_trf(uart1, config)

time.sleep(1.0)

setw.value = True
setw1.value = True

msg = [0x43, 0x44, 0x45, 0x46]
msg1 = [0x11, 0x22, 0x33]

uart.deinit()
uart1.deinit()
uart = busio.UART(tx=board.GP0, rx=board.GP1, baudrate=19200, timeout = 0.1)
uart1 = busio.UART(tx=board.GP4, rx=board.GP5, baudrate=19200, timeout = 0.1)

count = 0
while True:
    uart.write(bytes(msg))
    time.sleep(0.5)
    
    uart1.write(bytes(msg1))
    time.sleep(0.5)

    leng = uart.in_waiting
    if leng > 0:
        val = uart.read(leng)
        print_hex(val)
    leng1 = uart1.in_waiting
    if leng1 > 0:
        val1 = uart1.read(leng1)
        print_hex(val1)
    print()
        


```
