---
layout: post
title: Connecting to AWS IoT Core from Arduino
date: '2021-09-25'
author: Brian
tags: 
- AWS
- Arduino
- ESP32
- MQTT
- IoT
- TLS Mutual Authentication
---

Every Halloween my kids and I build some kind of decoration to scare everyone. The past few years we have been evolving a pneumatic wolf head that pops up and scares you. Then, it takes a picture of you looking silly. It was based on a RaspberryPi and AWS IoT. This year I wanted to move to Arduino, but I could not find instructions for connecting to AWS IoT from the Arduino.

AWS had an [SDK for Arduino](https://github.com/aws/aws-iot-device-sdk-arduino-yun) but it has not been maintained and does not appear to work with the ESP32 that I am using. There is also a [recent blog post for the ESP32](https://aws.amazon.com/blogs/compute/building-an-aws-iot-core-device-using-aws-serverless-and-an-esp32/) but I could not find the MQTT library they use from the Arduino IDE. I would like to use the [`ArduinoMqttClient`](https://github.com/arduino-libraries/ArduinoMqttClient) because it is actively maintained by Arduino and there are a lot of examples available for this library. However, there were no examples of using `ArduinoMqttClient` with AWS. 

AWS IoT Core supports [MQTT using TLS Mutual Authentication](https://docs.aws.amazon.com/iot/latest/developerguide/iot-authorization.html) with X.509 certificates for authentication. Most `ArduinoMqttClient` examples use basic auth with a username and password. I was able to build a solution by combining the above blog posts. 

First, we need a couple of libraries. The WiFi libraries should already be installed. You will need to add `ArduinoMqttClient`, but it is readily available from *Tools > Manage Libraries* menu. 

```c
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <ArduinoMqttClient.h>  
```

Next, we define some constants. I need the SSID and password to connect to WiFi. 

```c
// WiFi Info
const char WIFI_SSID[] = "MySSID";
const char WIFI_PASSWORD[] = "MyPassword";
```

I also need the AWS IoT endpoint and topic. You can get the endpoint from the AWS console and the topic can be anything you want. 

```c
// AWS IoT Core Connection Information
const char AWS_IOT_ENDPOINT[] = "##########.iot.us-east-1.amazonaws.com";
const char MQTT_TOPIC[]  = "MyTopic";
```

Finally, I need the TLS keys/certs. You can download these when you create an IoT thing from the AWS console. 

```c
// Amazon Root CA 1
static const char AWS_CERT_CA[] PROGMEM = R"EOF(
-----BEGIN CERTIFICATE-----
################################################################
-----END CERTIFICATE-----
)EOF";

// Device Certificate
static const char AWS_CERT_CRT[] PROGMEM = R"KEY(
-----BEGIN CERTIFICATE-----
################################################################
-----END CERTIFICATE-----
)KEY";

// Device Private Key
static const char AWS_CERT_PRIVATE[] PROGMEM = R"KEY(
-----BEGIN RSA PRIVATE KEY-----
################################################################
-----END RSA PRIVATE KEY-----
)KEY";
```

Now, we can create a WiFi and MQTT client. Most examples use the `WiFiClient`. I am using the 'WiFiClientSecure' which supports Mutual Authentication.

```c
WiFiClientSecure wifiClient = WiFiClientSecure();
MqttClient mqttClient(wifiClient);
```

In the Setup method, I first connect to WiFi. Then, I connect to AWS IoT and subscribe to the MQTT topic. 

```c
void setup()
{

  // initialize the serial port
  Serial.begin(115200);

  // Connect to Wi-Fi network with SSID and password
  Serial.print("Connecting to ");
  Serial.println(WIFI_SSID);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  // Print local IP address 
  Serial.println("");
  Serial.println("WiFi connected.");
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP());

  // Connect to AWS IoT
  wifiClient.setCACert(AWS_CERT_CA);
  wifiClient.setCertificate(AWS_CERT_CRT);
  wifiClient.setPrivateKey(AWS_CERT_PRIVATE);
  if (mqttClient.connect(AWS_IOT_ENDPOINT, 8883)) {
    Serial.println("You're connected to the MQTT broker!");
    Serial.println();
  } else {
    Serial.print("MQTT connection failed! Error code = ");
    Serial.println(mqttClient.connectError());
  }

  // Subscribe to MQTT and register a callback
  mqttClient.onMessage(messageHandler);
  mqttClient.subscribe(MQTT_TOPIC);
}
```

My message subscribe handler prints the message to the serial console.

```c
void messageHandler(int messageSize) {
  // we received a message, print out the topic and contents
  Serial.print("Received a message with topic '");
  Serial.print(mqttClient.messageTopic());

  // use the Stream interface to print the contents
  while (mqttClient.available()) {
    Serial.print((char)mqttClient.read());
  }
  
  Serial.println("");
}
```

And my loop simply calls `mqttClient.Poll()`.

```
void loop()
{
  mqttClient.poll();
}
```

That's all you need. 