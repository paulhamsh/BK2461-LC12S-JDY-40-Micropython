# BK2461-LC12S-JDY-40-Micropython
Micropython to control the LC12S and JDY-40 wireless communication boards based on the BK2461


```
LC12S datasheet     https://arduinolab.pw/wp-content/uploads/2019/05/H2-LCS12.pdf
                    http://user.cavenet.com/jgurley/lc12s/LC12S_datasheet.pdf

JDY-40 datasheet    https://w.electrodragon.com/w/images/0/05/EY-40_English_manual.pdf

BK2461 datasheet    https://www.mikrocontroller.net/attachment/381910/BT-WiFi-52rf5541.pdf

Pan 7420 datasheet  https://bbs.panchip.com/forum.php?mod=attachment&aid=MTkwNnxkYzA2YTQxZnwxNzI5Nzg0NTc3fDB8Nzk3Mw%3D%3D
```

The LC12S seems to have a historic blue version but can only seem to be purchased as the red v2 version (in 2024).  This uses a BK2461.   
JDY-40 has two versions, which are not compatible over wireless. The UART interface and AT commands seem similar. v1 uses the BK2461 chip and v2 uses a Panchip 7420.    

## LC12S configuration

Byte       |  Value       | Description
-----------|--------------|---------------
1          |  0x44, 0x5a  |    Header
3          |  0x22, 0x44  |    Device Id (cannot be altered on LC12S)
5          |  0x11, 0x33  |    Network address
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

Configuration is always at 9600 baud.    

Configuration is not remembered between power on cycles.    

## JDY-4 configuration

Command        |  Description
---------------|----------------
AT+BAUD\r\n    |            
AT+RFID\r\n    |  
AT+DVID\r\n    |  
AT+RFC\r\n     |  
AT+CLSS\r\n    |

Command            |  Description
-------------------|----------------
AT+DVID2244\r\n    |  
AT+RFID1133\r\n    |  
AT+RFC001\r\n      |  
AT+BAUD6\r\n       |  
  


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

## One pico running two boards - select JDY or LC12S

```
import board
import digitalio
import busio
from digitalio import DigitalInOut, Direction, Pull
import time

uart1_type = "JDY" #"LC12S"
uart0_type = "LC12S" #"LC12S"
DEV = 2 #2

if DEV == 1:
    msg0 = [0x10, 0x11, 0x12, 0x13]
    msg1 = [0x20, 0x21, 0x22, 0x23]
else:
    msg0 = [0x40, 0x41, 0x42, 0x43]
    msg1 = [0x50, 0x51, 0x52, 0x53]    

def print_hex(val):
    print("".join(" x%02x" % i for i in val))

def uart_bin_trf(uart, cmd):
    buf = bytes(cmd)
    print_hex(buf)
    uart.write(buf)
    time.sleep(0.5)
    leng = uart.in_waiting
    if leng > 0:
        val = uart.read(leng)
        print_hex(val)
    print()

def uart_str_trf(uart, cmd):
    buf = bytes(cmd)
    print(buf)
    uart.write(buf)
    time.sleep(0.5)
    leng = uart.in_waiting
    if leng > 0:
        val = uart.read(leng)
        print(val)
    print()

setw0 = DigitalInOut(board.GP2)
setw0.direction = Direction.OUTPUT
csw0=DigitalInOut(board.GP3)
csw0.direction = Direction.OUTPUT

setw1 = DigitalInOut(board.GP6)
setw1.direction = Direction.OUTPUT
csw1=DigitalInOut(board.GP7)
csw1.direction = Direction.OUTPUT

setw0.value = True
csw0.value = True
setw1.value = True
csw1.value = True

time.sleep(1.0)

setw0.value = False
csw0.value = False
setw1.value = False
csw1.value = False

time.sleep(1.0)


lc12s_config = [0xAA, 0x5A, 0x22, 0x44, 0x11, 0x33, 0x00, 0x06, 0x00, 0x05, 0x00, 0x01, 0x00, 0x00, 0x00, 0x12, 0x00, 0x2e]
temp = 0
for i in range(17):
    temp += lc12s_config[i]
lc12s_config[17] = temp % 256

def configure(uart_name, uart_type, tx_pin, rx_pin, baud):
    print("Configure %s as %s" %(uart_name, "LC12S" if uart_type == "LC12S" else "JDY"))
    print("-------------------")
    if uart_type== "LC12S":
        uart = busio.UART(tx=tx_pin, rx=rx_pin, baudrate=9600, timeout = 0.1)
        uart_bin_trf(uart, lc12s_config)
        uart.deinit()
        uart = busio.UART(tx=tx_pin, rx=rx_pin, baudrate=19200, timeout = 0.1)
    else:
        uart = busio.UART(tx=tx_pin, rx=rx_pin, baudrate=baud, timeout = 0.1)
        uart_str_trf(uart, b"AT+DVID2244\r\n")
        uart_str_trf(uart, b"AT+RFID1133\r\n")
        uart_str_trf(uart, b"AT+RFC001\r\n") 
        uart_str_trf(uart, b"AT+BAUD6\r\n")
        
        if baud != 19200:
            uart.deinit()            
            uart = busio.UART(tx=tx_pin, rx=rx_pin, baudrate=19200, timeout = 0.1)
    return uart
    
uart0 = configure("uart0", uart0_type, board.GP0, board.GP1, 19200)    
uart1 = configure("uart1", uart1_type, board.GP4, board.GP5, 19200)    

time.sleep(1.0)

setw0.value = True
setw1.value = True

print("Start messaging")
print("---------------")

while True:
    time.sleep(0.3)

    uart0.write(bytes(msg0))
    time.sleep(0.3)

    leng = uart0.in_waiting
    if leng > 0:
        val = uart0.read(leng)
        print("Uart 0 %5s rx:" % uart0_type, end="")
        print_hex(val)
    
    time.sleep(0.3)
    uart1.write(bytes(msg1))
    time.sleep(0.3)
    
    leng1 = uart1.in_waiting
    if leng1 > 0:
        val1 = uart1.read(leng1)
        print("Uart 1 %5s rx:"% uart1_type, end="")
        print_hex(val1)
    
    print()
        
```
