---
title: RoboCam Project
tags: Roboterra
---
We built a cart that was able to point at one's face and move forward during our intern at Roboterra. The project was based on Raspberry Pi and Face++. 

![robocam](/pic/robocam.png)



<!-- more -->

## **System Implementation**
### **1. setup**
#### **hardware:**
* Raspberry Pi B+
* Pi Camera
* Robocore
* Servo × 2
* Motor × 2
* Origin Kit
* 5v battery

[Written by Li]

#### **software**
1. When you buy a raspberrypi, it's just a bare board without any operating system. Let's assume you have already installed latest Raspbian OS on your raspberrypi and and have succesfully remote control your pi. If you haven't yet finished, please follow the instruction below to **initialize your raspberrypi**:
   http://yesyzq.github.io/2016/01/09/raspberry/
   Make sure that you have network environment for raspberry pi to work properly.

2. Go to http://www.faceplusplus.com/ and get an account from that website. Follow the instructions of **Face++** to setup a local API on your pi. 
   In case you are not familiar with **Linux command**:
   [here](http://yesyzq.github.io/2016/01/17/Linux%E5%91%BD%E4%BB%A4%E4%BB%A5%E5%8F%8A%E5%BF%AB%E6%8D%B7%E9%94%AE%E6%95%B4%E7%90%86/) 
   You may also refer to any other guide online. There are always abundant resources waiting for you to explore on the internet.

3. Python2 is pre-installed in Raspbian, is you find no python in your OS, try `sudo apt-get install python-dev` in the console. If it does not work, click here:
   http://www.eeboard.com/bbs/forum.php?mod=viewthread&tid=1815

Then you get everything ready to build the project. Have fun exploring!

[Written by Yu]
### **2. mechanical design**
**Top view:**
![Top view](/pic/3-31-2.JPG)
**Bottom view:**
![Bottom view](/pic/3-31-3.PNG)
**Side view:**
![Side view](/pic/3-31-4.PNG)

[Written by Li]
### **3. communication**
#### **3.1 communication between pi and Face++ server**
First of all, you need to capture the image using picamera. Create a python file, type in the following command:
`os.system('raspistill -o test.jpg -t 1 -w 300 -h 224 -q 50')`
Feel free to try it out in the console first to see how it works. For detailed information about the command `raspistill`, click here:
http://www.ithao123.cn/content-1549646.html

After capturing the image, you need to send the image to the server in order to extract the information in which the coordinates of faces are located.
In the python file, type in the following command:
`tmp = api.detection.detect(img = File(r'hellow.jpg'))`
Read the coordinate of the center of the face
`cent=tmp["face"][0]["position"]["center"]["x"]`

[Written by Yu]
#### **3.2 communication between pi and RoboCore**
We use serial communication to send message from Raspberry Pi to Robocore. Code on Robocore should look like this:
```C
#include <Servo.h>
Servo myservo;

byte number = 0;
int angle=90;
void setup(){
     Serial.begin(9600);
     myservo.attach(9);
     myservo.write(angle);
}

void loop(){
    if (Serial.available()) {
       number = 0;
       number = Serial.read();
       if(number == B01101100){ //if number == 'l'
          angle = angle -5;
       }
       else if(number == B01110010){ //if number == 'r'
          angle = angle +5;
       }
       myservo.write(angle);
   }
   
}
```
Every time when Robocore receives 'l', the servo will turn left by 5 degree. When it receive 'r', the servo do te opposite. 

[Written by Li]

### **4. Main code**
```python
import os
os.system('sh run.sh') #improt Face ++ apis
import os
import serial
import threading
import time
from time import sleep, ctime
ser = serial.Serial('/dev/ttyUSB0', 9600, timeout=1) #need to use this command: ls /dev/tty* to see Robocore is on USB0 or USB1.
ser.open()

last = 3 #position of the face in last detection
cur = 3  #position of the face in current detection

def compress(x):  #classify the position of the face
	if(x>=40) and (x<60):
		return 3
	elif(x>=20) and (x<40):
		return 2
	elif(x>=0) and (x<20):
		return 1
	elif(x>=60) and (x<80):
		return 4
	elif(x>=80) and (x<=100):
		return 5

def fun_turn(turn): #serial communication with robocore
	if turn==1:
		ser.write("l")
	elif turn==2:
		ser.write("ll")
	elif turn==3:
		ser.write("lll")
	elif turn==4:
		ser.write("llll")
	elif turn==-1:
		ser.write("r")
	elif turn==-2:
		ser.write("rr")
	elif turn==-3:
		ser.write("rrr")
	elif turn==-4:
		ser.write("rrrr")
	else:
		ser.write("")

while 1:
	try:
		os.system('raspistill -o hellow.jpg -t 1 -w 300 -h 224 -q 50') #take picture
		tmp = api.detection.detect(img = File(r'hellow.jpg')) #send requirement to Face++
		try:
			cent=tmp["face"][0]["position"]["center"]["x"]
		except IndexError: #in case there is no face
			continue
		cent
		cur = compress(cent)
		turn = cur - last
		fun_turn(turn)
		last = cur
	except KeyboardInterrupt: #to exit
		break
```
**How to use the code:**
After you have got the Face++ API files, you should put them in one folder. Then enter that folder by using command `cd`. Then you should type `python` and paste the main code into it. 

[Written by Li]

### **5. Further Development**
However, such kind of implementation is not fast enough to do real-time detection and tracking. There are several ways to improve the current implementation:

**5.1 Localize**
In face we do not need to rely on the server to do the detection. Since picamera itself can capture OpenCV image(yuv image), it might be faster if we install openCV python on raspberry pi directly and build the detection algorithm locally. (need to solve this problem: Li cannot install OpenCV on his Pi following this instruction:
http://www.pyimagesearch.com/2015/02/23/install-opencv-and-python-on-your-raspberry-pi-2-and-b/)

**5.2	Multi-thread**
Multi-threads of the program to utilize the waiting time of Face++ api (need to solve this problem: when two program run `raspistill` at the same time, the camera will be disabled until next reboot)

[Written by Yu]

### 6. Demo

<video controls style="width: 100%"><source src="/video/cam.mp4" type="video/mp4"></video>

### 7. Team Work Distribution**

**Yu Zheqing:**
* Came up with the whole structure of the system
* Set up Face++ API in Raspberry Pi 

**Li Zihan:**
* Built the solid structure
* Wrote the main python program and the Robocore program
* Came up with two ways to optimize the system but none of them works
