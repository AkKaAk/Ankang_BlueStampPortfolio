# Wrist Rehab Device
My project aims to build a wrist rehabilitation device that not only tracks when a user's wrist position is not ideal, but also tracks whether they are doing rehab exercises with proper form using the accelerometer and gyroscope. It also tracks their rehab process by using the flex sensor to track their range of motion (ROM—the angle that their wrist can bend or twist in a certain direction) and letting users see how they've improved.

<!---
Replace this text with a brief description (2-3 sentences) of your project. This description should draw the reader in and make them interested in what you've built. You can include what the biggest challenges, takeaways, and triumphs from completing the project were. As you complete your portfolio, remember your audience is less familiar than you are with all that your project entails!

You should comment out all portions of your portfolio that you have not completed yet, as well as any instructions:
-->
<!---
HTML 
<!--- This is an HTML comment in Markdown -->
<!--- Anything between these symbols will not render on the published site -->

| **Engineer** | **School** | **Area of Interest** | **Grade** |
|:--:|:--:|:--:|:--:|
| Ankang H | Archbishop Mitty High School | Mechanical Engineering | Incoming Senior

<!---
**Replace the BlueStamp logo below with an image of yourself and your completed project. Follow the guide [here](https://tomcam.github.io/least-github-pages/adding-images-github-pages-site.html) if you need help.**

![Headstone Image](logo.svg)
-->
# Final Milestone
<!---
**Don't forget to replace the text below with the embedding for your milestone video. Go to Youtube, click Share -> Embed, and copy and paste the code to replace what's below.**

<iframe width="560" height="315" src="https://www.youtube.com/embed/F7M7imOVGug" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

For your final milestone, explain the outcome of your project. Key details to include are:
- What you've accomplished since your previous milestone
- What your biggest challenges and triumphs were at BSE
- A summary of key topics you learned about
- What you hope to learn in the future after everything you've learned at BSE
-->

# Second Milestone
<!---
**Don't forget to replace the text below with the embedding for your milestone video. Go to Youtube, click Share -> Embed, and copy and paste the code to replace what's below.**

<iframe width="560" height="315" src="https://www.youtube.com/embed/y3VAmNlER5Y" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

For your second milestone, explain what you've worked on since your previous milestone. You can highlight:
- Technical details of what you've accomplished and how they contribute to the final goal
- What has been surprising about the project so far
- Previous challenges you faced that you overcame
- What needs to be completed before your final milestone
-->

# First Milestone
<!---
**Don't forget to replace the text below with the embedding for your milestone video. Go to Youtube, click Share -> Embed, and copy and paste the code to replace what's below.**

<iframe width="560" height="315" src="https://www.youtube.com/embed/CaCazFBhYKs" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

For your first milestone, describe what your project is and how you plan to build it. You can include:
- An explanation about the different components of your project and how they will all integrate together
- Technical progress you've made so far
- Challenges you're facing and solving in your future milestones
- What your plan is to complete your project
-->

# Starter Project - Jitterbug

<iframe width="560" height="315" src="https://www.youtube.com/embed/nMMjlUX_D_o?si=9CbDTJQDhTlcbcu3" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

# Schematics
<!---
```Here's where you'll put images of your schematics. [Tinkercad](https://www.tinkercad.com/blog/official-guide-to-tinkercad-circuits) and [Fritzing](https://fritzing.org/learning/) are both great resoruces to create professional schematic diagrams, though BSE recommends Tinkercad becuase it can be done easily and for free in the browser.```
-->

![Wiring Schematic](wrist_device_schematic.png)

# Milestone 1 Code

```c++
#include <BLEDevice.h>
#include <BLEServer.h>
#include <BLEUtils.h>
#include <BLE2902.h>
#include <Adafruit_LSM6DS3TRC.h>
#include <Adafruit_LIS3MDL.h>
#include <Adafruit_AHRS.h>

#define SERVICE_UUID           "6E400001-B5A3-F393-E0A9-E50E24DCCA9E"
#define RX_CHARACTERISTIC_UUID "6E400002-B5A3-F393-E0A9-E50E24DCCA9E"
#define TX_CHARACTERISTIC_UUID "6E400003-B5A3-F393-E0A9-E50E24DCCA9E"

BLEServer *pServer = NULL;
BLECharacteristic *pTxCharacteristic;
bool deviceConnected = false;

Adafruit_LSM6DS3TRC lsm6ds;
Adafruit_LIS3MDL lis3mdl;
Adafruit_NXPSensorFusion filter;

const int FLEX_PIN = 32;
const int buzzer = 23;

unsigned long lastPrintTime = 0;
unsigned long lastFilterUpdate = 0;
unsigned long buzzerCycleTime = 0;

sensors_event_t accel, gyro, mag, temp;

float accelX = 0, accelY = 0, accelZ = 0;
float tareX = 0, tareY = 0, tareZ = 0;
bool isCalibrated = false;

// Defines offset for magnetometer
#define MAG_HARD_IRON_X -44.56
#define MAG_HARD_IRON_Y 33.52
#define MAG_HARD_IRON_Z 7.34

class MyServerCallbacks: public BLEServerCallbacks {
    void onConnect(BLEServer* pServer) {
      deviceConnected = true;
    };
    void onDisconnect(BLEServer* pServer) {
      deviceConnected = false;
      pServer->getAdvertising()->start(); // Restart advertising so ESP-32 can reconnect without resetting the board
    }
};

void setup() {
  Serial.begin(9600);

  Wire.begin();
  
  BLEDevice::init("AK_ESP32_Bluetooth");
  pServer = BLEDevice::createServer();
  pServer->setCallbacks(new MyServerCallbacks());

  BLEService *pService = pServer->createService(SERVICE_UUID);

  pTxCharacteristic = pService->createCharacteristic(TX_CHARACTERISTIC_UUID, BLECharacteristic::PROPERTY_NOTIFY);
  pTxCharacteristic->addDescriptor(new BLE2902());

  BLECharacteristic *pRxCharacteristic = pService->createCharacteristic(RX_CHARACTERISTIC_UUID, BLECharacteristic::PROPERTY_WRITE);

  pService->start();
  pServer->getAdvertising()->start();
  Serial.println("BLE UART is ready and advertising!");
  
  lsm6ds.begin_I2C();
  lis3mdl.begin_I2C();
  lsm6ds.setAccelDataRate(LSM6DS_RATE_208_HZ);
  lsm6ds.setGyroDataRate(LSM6DS_RATE_208_HZ);
  filter.begin(200); // Makes the sensor fusion algorithm make a calculation every 100 Hz

  pinMode(FLEX_PIN, INPUT);
  pinMode(buzzer, OUTPUT);
}

void loop() {
  buzzerCycleTime = millis() % 2500;

  if (millis() - lastFilterUpdate >= 5) {
    lastFilterUpdate += 5;

    lsm6ds.getEvent(&accel, &gyro, &temp);
    lis3mdl.getEvent(&mag);

    filter.update(gyro.gyro.x * 180.0 / PI, gyro.gyro.y * 180.0 / PI, gyro.gyro.z * 180.0 / PI, accel.acceleration.x / 9.80665, accel.acceleration.y / 9.80665, accel.acceleration.z / 9.80665, mag.magnetic.x - MAG_HARD_IRON_X, mag.magnetic.y - MAG_HARD_IRON_Y, mag.magnetic.z - MAG_HARD_IRON_Z);
    filter.getLinearAcceleration(&accelX, &accelY, &accelZ);
  }

  if (!isCalibrated && millis() >= 3000) {
    tareX = accelX * 9.80665;
    tareY = accelY * 9.80665;
    tareZ = accelZ * 9.80665;
    isCalibrated = true;
  }

  if (millis() - lastPrintTime >= 500) {
    lastPrintTime = millis();

    int flexValue = analogRead(FLEX_PIN);

    float cleanX = accelX * 9.80665 - tareX;
    float cleanY = accelY * 9.80665 - tareY;
    float cleanZ = accelZ * 9.80665 - tareZ;

    Serial.println("Flex: " + String(flexValue));

    Serial.print("Accel X: "); Serial.print(cleanX);
    Serial.print(" | Y: ");    Serial.print(cleanY);
    Serial.print(" | Z: ");    Serial.print(cleanZ);
    Serial.println(" m/s^2");

    Serial.print("Gyro X: ");  Serial.print(gyro.gyro.x);
    Serial.print(" | Y: ");    Serial.print(gyro.gyro.y);
    Serial.print(" | Z: ");    Serial.print(gyro.gyro.z);
    Serial.println(" rad/s");

    String bluetoothData = "";

    bluetoothData += ("Flex: " + String(flexValue) + "\nAccel X: " + String(cleanX) + " | Y: " + String(cleanY) + " | Z: " + String(cleanZ) + " m/s^2\nGyro X: " + String(gyro.gyro.x) + " | Y: " + String(gyro.gyro.y) + " | Z: " + String(gyro.gyro.z) + " rad/s\n");

    if (deviceConnected) {
      pTxCharacteristic->setValue(String(bluetoothData).c_str());
      pTxCharacteristic->notify(); // Pushes data out immediately
    }
  }
  
  static int buzzerMode = 0;
  // Makes buzzer beep at 6 kHz for 175 ms then 50 Hz for 175 ms 4 times, then repeats this sequence after 1100 ms of silence
  if (buzzerCycleTime < 1400) {
    if (buzzerCycleTime % 350 < 175) {
      if (buzzerMode != 1) {
        tone(buzzer, 6000, 175);
        buzzerMode = 1;
      }
    } else {
      if (buzzerMode != 2) {
        tone(buzzer, 50, 175);
        buzzerMode = 2;
      }
    }
  } else {
    if (buzzerMode != 0) {
      noTone(buzzer);
      buzzerMode = 0;
    }
  }

  yield();
}
```

# Bill of Materials
<!---
Here's where you'll list the parts in your project. To add more rows, just copy and paste the example rows below.
Don't forget to place the link of where to buy each component inside the quotation marks in the corresponding row after href =. Follow the guide [here]([url](https://www.markdownguide.org/extended-syntax/)) to learn how to customize this to your project needs.

| **Part** | **Note** | **Price** | **Link** |
|:--:|:--:|:--:|:--:|
| Item Name | What the item is used for | $Price | <a href="https://www.amazon.com/Arduino-A000066-ARDUINO-UNO-R3/dp/B008GRTSV6/"> Link </a> |
| Item Name | What the item is used for | $Price | <a href="https://www.amazon.com/Arduino-A000066-ARDUINO-UNO-R3/dp/B008GRTSV6/"> Link </a> |
| Item Name | What the item is used for | $Price | <a href="https://www.amazon.com/Arduino-A000066-ARDUINO-UNO-R3/dp/B008GRTSV6/"> Link </a> |
-->

# Other Resources/Examples
<!---
One of the best parts about Github is that you can view how other people set up their own work. Here are some past BSE portfolios that are awesome examples. You can view how they set up their portfolio, and you can view their index.md files to understand how they implemented different portfolio components.
- [Example 1](https://trashytuber.github.io/YimingJiaBlueStamp/)
- [Example 2](https://sviatil0.github.io/Sviatoslav_BSE/)
- [Example 3](https://arneshkumar.github.io/arneshbluestamp/)

To watch the BSE tutorial on how to create a portfolio, click here.
-->
