# OpenMV

P4 TXD

P5 RXD

从VIN处上电，不要从3.3V排线处施加外部电压，会烧毁OpenMV

```python
import sensor, image, time,math,pyb
from pyb import UART,LED
import json
import ustruct

sensor.reset()
sensor.set_pixformat(sensor.RGB565)
sensor.set_framesize(sensor.QVGA)
sensor.skip_frames(time = 2000)
sensor.set_auto_gain(False) # must be turned off for color tracking
sensor.set_auto_whitebal(False) # must be turned off for color tracking
red_threshold_01=(10, 100, 127, 32, -43, 67)
clock = time.clock()

uart = UART(3,115200)   #定义串口3变量
uart.init(115200, bits=8, parity=None, stop=1) # init with given parameters

led = pyb.LED(1)# Red LED = 1, Green LED = 2, Blue LED = 3, IR LEDs = 4.

def find_max(blobs):    #定义寻找色块面积最大的函数
    max_size=0
    for blob in blobs:
        if blob.pixels() > max_size:
            max_blob=blob
            max_size = blob.pixels()
    return max_blob


def sending_data(cx,cy,cw,ch):
    global uart;
    #frame=[0x2C,18,cx%0xff,int(cx/0xff),cy%0xff,int(cy/0xff),0x5B];
    #data = bytearray(frame)
    data = ustruct.pack("<bbhhhhb",      #格式为俩个字符俩个短整型(2字节)
                   0x2C,                      #帧头1
                   0x12,                      #帧头2
                   int(cx), # up sample by 4   #数据1
                   int(cy), # up sample by 4    #数据2
                   int(cw), # up sample by 4    #数据1
                   int(ch), # up sample by 4    #数据2
                   0x5B)
    uart.write(data);   #必须要传入一个字节数组


while(True):
    clock.tick()
    led.off()
    img = sensor.snapshot()
    blobs = img.find_blobs([red_threshold_01])
    cx=0;cy=0;
    if blobs:
            max_b = find_max(blobs)
            #如果找到了目标颜色
            cx=max_b[5]
            cy=max_b[6]
            cw=max_b[2]
            ch=max_b[3]
            img.draw_rectangle(max_b[0:4]) # rect
            img.draw_cross(max_b[5], max_b[6]) # cx, cy
            FH = bytearray([0x2C,0x12,cx,cy,cw,ch,0x5B])
            #sending_data(cx,cy,cw,ch)
            #print(FH)
            uart.write(FH)
            print(cx,cy,cw,ch)
            led.on()
            time.sleep_ms(1000)


```

