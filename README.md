# Triangulation algorithm
Triangulation for estimating robot position using stepper motor, photoelectric retro-reflective sensor and a gyroscope for Arduino.
## Components used
* 1x [Arduino UNO](https://store.arduino.cc/arduino-uno-smd-rev3)
* 1x 42BYGHM809 Stepper Motor
* 1x [A4988 Stepper Motor Driver](https://www.pololu.com/product/1182)
* 1x Sick WL280-2 Photoelectric retro-reflective sensor
* 1x MPU6050 Gryroscope and Accelerometer sensor
* 1x 4N25/4N35 Optocoupler
* 1x 9V-30V Power source
* 2x 1k Ohm resistors
* 3x Beacons with reflective bands on them
* 1x Breadboard

## Arduino libraries
* [I2Cdev](https://www.i2cdevlib.com/)
* MPU6050 (*included in I2Cdev*)

## Laser Wiring
![wiring](wiring.png)

* Connect the photoelectric sensor to the 9V-30V power source
* The output of the photoelectric sensor goes into the 4N25/4N35 anode. The photoelectric sensor has inverted logic: Vcc when is not reflecting, GND when it's reflecting.
* Yellow wire is the laser interrupt pin (location can be changed in code)

## Stepper motor wiring
![wiring](https://cdn.instructables.com/FOE/ACRR/IDFSFSP3/FOEACRRIDFSFSP3.LARGE.jpg)

[Source for image](http://www.instructables.com/id/Drive-a-Stepper-Motor-with-an-Arduino-and-a-A4988-/)

The source for image above can be used to connect the driver to the stepper motor and to the Arduino. The only difference is some connections:
 * *Step* pin from driver goes into Arduino's pin 4 (can be modified in code)
 * *Dir* pin from driver goes into Arduino's pin 5 (can be modified in code)

 ## Gyroscope wiring
 ![wiring](https://maker.pro/storage/LuePqRP/LuePqRP4SZEwZVOOiKOtAwz8yLXRET6Z3Bql0M33.png)

 [Source for image](https://maker.pro/arduino/tutorial/how-to-interface-arduino-and-the-mpu-6050-sensor)

 Again, the link above can be used to connect the MPU6050 gyroscope to the Arduino. The only difference is that the *INT* pin from MPU6050 goes into Arduino's pin 3 (can be modified in code).

## Triangulation algorithm
The triangulation method and algorithms can be found [here](http://www.telecom.ulg.ac.be/triangulation/). In this example, I used the **ToTal** algorithm. The main problem of this algorithm is that you need to know which angle corresponds to which beacon. Without appropiate correction, this works only when the robot is moving straight, without rotation. To fix this, a gyroscope is used to correct the stepper motor position such that it always starts from the same orientation, thus knowing that the first angle corresponds to the first beacon.

For this specific stepper motor, one step means 0.9 degrees. Using the photoelectric laser mounted on top of the motor, when it reflects in any of the beacons, it generates an interrupt on the pin 2. In the interrupt routine, the angle is calculated and saved.

```
angle = steps * 0.9
```

Also, the angles are counted. If there is any reflective material on scene or some beacon was missed, the triangulation will not be done.

To fix the problem mentioned above, each time a rotation is done, the next rotation is corrected by the yaw:

```
rotationSteps = 400 - ((lastCorrectionYaw - yaw) / 0.9f);
lastCorrectionYaw = yaw;
```
Here, 400 steps means a complete rotation (360 degrees). 0.9 is used to convert from degrees to steps. In the code, you will also see the handling of jumps from 360 degrees to 0 and from 0 to 360.

## Gyroscope filtering
The MPU6050 is known to have a drift. The MPU on it is able to filter it using the accelerometer, but it can't do it instantly. Until it reaches stability it may take some time. Also, the value will not go to 0, but it will reach stability at some degree. 

An initialization routine exists such that it saves that angle as an offset angle, and the next measurements have this angle substracted. The first interrupt generated on the laser interrupt pin will execute this routine. You have to generate this manually when the value is stable by applying ground to the optocoupler anode.

This interrupt will also start the stepper motor.

## TODO
* The laser does not always reflect on the same point on beacon, so the actual locations may differ a bit. Find a way to fix this.
    * **Hint 1**: Take also the FALLING angle and do the mean between them?
    * **Hint 2**: Sum all the RISING and FALLING angles from a beacon and do the mean?
* More tests. The correction of the robot rotation may fail sometimes.
* Filter resulted positions from wrong angles (when the robot was rotating).