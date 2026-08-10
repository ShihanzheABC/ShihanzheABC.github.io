---
title: "Hardware Prototyping and System Automation"
excerpt: "Automated 2D Scanning and Real-Time Imaging Platform for Scalable Arrays<br/><img src='/images/fig7.jpg'>"
collection: portfolio
---

## Project Overview
In this project, I engineered an automated multi-channel data acquisition system designed for the real-time imaging of organic photodetector (OPD) arrays. The system bridges custom-built analog circuits with digital processing to enable high-throughput optoelectronic characterization.

## 4*4 real-time image system
* **Arduino Uno:**
* 
    //Arduino Uno   
const int selectPins[4] = {2, 3, 4, 5};  
void setup() {  
  Serial.begin(115200);  
  for (int i = 0; i < 4; i++)   
  pinMode(selectPins[i], OUTPUT);  
}
void loop() {  
  if (Serial.available() > 0) {  
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

#python code
import serial  
import pyvisa  
import time  
import numpy as np  
import csv  
import matplotlib  
matplotlib.use('TkAgg')    
import matplotlib.pyplot as plt  

#1. 端口与文件名配置  
ARDUINO_PORT = 'COM10'
KEITHLEY_PORT = 'COM12'
#自动生成以时间命名的文件名
DATA_FILE = f"ScanData_{time.strftime('%Y%m%d_%H%M%S')}.csv"

#2. 初始化硬件连接  
ser = serial.Serial(ARDUINO_PORT, 115200, timeout=0.1)  # 缩短串口超时
rm = pyvisa.ResourceManager()
keithley = rm.open_resource(f'ASRL{KEITHLEY_PORT[3:]}::INSTR',
                            read_termination='\r',
                            write_termination='\r',
                            timeout=1000)
def setup_turbo_keithley():
    keithley.write("*RST")
    keithley.write(":FORM:ELEM CURR")
    keithley.write(":SOUR:FUNC VOLT")
    keithley.write(":SOUR:VOLT 0.0")
    keithley.write(":SENS:FUNC 'CURR'")

 #提速关键指令 ---
    keithley.write(":SENS:CURR:NPLC 0.01")  # 极速采样模式
    keithley.write(":SENS:CURR:RANG 10e-6")  # 根据你的器件电流固定量程(例如10uA)
    keithley.write(":DISP:ENAB OFF")  # 关闭源表面板显示，大幅提速
    keithley.write(":OUTP ON")
    print("源表已进入极速采集模式 (面板显示已关闭)")
setup_turbo_keithley()  
#3. 初始化 CSV 文件 ---
with open(DATA_FILE, 'w', newline='') as f:
    writer = csv.writer(f)
    # 表头：时间戳 + 16个像素点
    header = ['Timestamp'] + [f'Pixel_{i}' for i in range(16)]
    writer.writerow(header)
#4. 实时成像初始化 ---
plt.ion()
fig, ax = plt.subplots(figsize=(6, 5))
data_matrix = np.zeros((4, 4))
im = ax.imshow(data_matrix, cmap='inferno', interpolation='gaussian', origin='lower')
plt.colorbar(im, label='Current (A)')
ax.set_title("Ultra-Fast 4x4 Imaging & Data Saving")  
#5. 极速扫描与保存函数 ---  
def scan_and_save():
    current_frame = []
    timestamp = time.time()
    ser.reset_input_buffer()

    for i in range(16):
        ser.write(bytes([i]))
        # 极短等待确认  
        if ser.read(1):
            try:
                res = keithley.query(":READ?")
                val = float(res.strip().split(',')[0].replace('A', ''))
                current_frame.append(val)
            except:
                current_frame.append(0.0)
        else:
            current_frame.append(0.0)

    # 补齐数据  
    while len(current_frame) < 16: current_frame.append(0.0)
    final_data = current_frame[:16]

    # 实时保存 
    with open(DATA_FILE, 'a', newline='') as f:
        writer = csv.writer(f)
        writer.writerow([timestamp] + final_data)

    return np.array(final_data).reshape(4, 4)


#6. 主循环
try:
    print(f"数据将实时保存至: {DATA_FILE}")
    while True:
        data = scan_and_save()

        # 更新图像
        im.set_data(data)
        v_min, v_max = data.min(), data.max()
        im.set_clim(vmin=v_min, vmax=v_max + 1e-15)

        # 强制快速刷新
        fig.canvas.draw_idle()
        fig.canvas.flush_events()
        # 移除控制台繁琐打印，改为极简显示
        print(f"FPS: {1 / (time.time() - time.time()):.1f} | Peak: {v_max:.2e} A", end='\r')

except KeyboardInterrupt:
    print("\n采集已停止")
finally:
    keithley.write(":DISP:ENAB ON")  # 恢复源表显示
    keithley.write(":OUTP OFF")
    keithley.close()
    ser.close()
    print(f"所有数据已安全存入 {DATA_FILE}")
