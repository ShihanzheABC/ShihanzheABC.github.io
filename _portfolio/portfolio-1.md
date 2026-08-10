---
title: "Hardware Prototyping and System Automation"
excerpt: "Automated 2D Scanning and Real-Time Imaging Platform for Scalable Arrays<br/><img src='/images/fig7.jpg'>"
collection: portfolio
---

## Project Overview
In this project, I engineered an automated multi-channel data acquisition system designed for the real-time imaging of organic photodetector (OPD) arrays. The system bridges custom-built analog circuits with digital processing to enable high-throughput optoelectronic characterization.

## 4*4 real-time image system
* **Arduino Uno:**

const int selectPins[4] = {2, 3, 4, 5};  

void setup() {  

  Serial.begin(115200);  
  
  for (int i = 0; i < 4; i++)   
  
  pinMode(selectPins[i], OUTPUT);  
  
}

void loop() {  

  if (Serial.available() > 0) {  
  
    // 接收 PC 发来的目标通道字节 (0-15)  
    
    int targetChannel = Serial.read();   
    
    if (targetChannel >= 0 && targetChannel <= 15)  
    {  
    
      for (int i = 0; i < 4; i++)  
      {
      digitalWrite(selectPins[i], (targetChannel >> i) & 0x01);
      }
      Serial.write(1); // 发回确认信号，告知切换完成  
    }  
  }  
}  
  
* **python:**

