---
layout: post
title:  "Refactoring IMU Driver (Part 1): Start from scratch "
subtitle: "Why I start from scratch"
date:   2026-01-03 10:00:00 +0900
categories: [Robotics, ROS2, System Architecture]
tags: [C++, Optimization, Open Source]
description: "Vendor's code is legacy. WIT driver need to be modern"
---

# Modern ROS2 WIT IMU Driver DevLog - 1. Into the Wild

## 0. Intro: Back to the Wild

While refactoring and releasing the RPLIDAR driver previously, I realized one thing: people are just as interested in the **"struggles and problem-solving process"** as they are in the finished code.

This time, my target is **WIT MOTION's IMU**, a rising player in the industry. **While it offers an excellent price-to-performance ratio**, its support within the ROS2 environment is truly the "Wild West." In this series, I plan to document the process of taming this wild territory to build a robust driver that complies with **Modern C++ and ROS2 Lifecycle** standards.

---

## 1. Information Gathering: To Avoid Reinventing the Wheel

The first step in development is always preliminary research. If someone has already built the perfect driver, I could simply enjoy the luxury of a `git clone`.

### 1.1 Official Repository Analysis: Opening Pandora's Box

#### 1.1.1 The "Department Store" Strategy

Upon visiting the manufacturer's official GitHub, I didn't find a polished software engineering project, but rather a **"General Store"** approach.

![WIT sdk tree](/assets/img/2026-01-03-WIT-refactoring/department_store.png)

> **[🔗 GitHub Repository: WITMOTION/WitStandardProtocol_JY901](https://github.com/WITMOTION/WitStandardProtocol_JY901)**

Looking at the folder structure, the manufacturer's strategy is clear: _"We prepared everything because we didn't know what you would use."_ While this "kitchen sink" approach might be rational for a hardware manufacturer, for a robotics software engineer, it signals that **"nothing is maintained with any real depth."**

I focused on analyzing the core layers: `Serial` (Transport), `SDK` (Parser), and `ROS2 Node` (Application). Unfortunately, I discovered structural flaws that are unacceptable from an engineering standpoint.

#### 1.1.2 Transport Layer: A Relic from the 90s 

**Path: Linux_C/normal/serial.c**

First, I examined the serial communication code, which acts as the entry point for data. Surprisingly, it was not written in Modern C++, but preserved as a fossil of **1990s embedded C style**.

- **Busy Waiting (CPU Waste):** It is implemented using a polling method with `VTIME=0` and `VMIN=0`. This causes the CPU to consume cycles in an infinite loop even when no data is available. It is like putting a tractor engine in a high-performance robot PC.
    
- **Lack of RAII:** Resource management is manual. The developer must manually handle `open` and `close` functions, and there is virtually no exception handling.
    
- **Hardcoded Settings:** Configuration options, such as Baudrate, are hardcoded, offering zero scalability.
    

#### 1.1.3 SDK Layer: A Feast of Global Variables 

**Path: Linux_C/normal/wit_c_sdk.c**

The SDK layer for parsing data is even more concerning. It has a structure that directly contradicts **Object-Oriented Programming (OOP)** principles.

- **Lack of Thread-Safety:** All key data buffers are declared as `static` global variables. This implies that the driver supports **"only one sensor per process."** If you were to mount two IMUs on a robot, the data would overlap, likely causing a system crash.
    
- **Callback Issues:** It uses pure C function pointers without considering C++ class member functions. This makes integration into modern C++ projects very difficult.
    

#### 1.1.4 Application Layer: ROS1 Code Disguised as ROS2 

**Path: ROS/wit/wit_ros_ws/src/scripts/wit_normal_ros.py**

The most shocking part was the Python script provided as a "ROS2 Node." While the filename said ROS2, the internal code was full of legacy patterns from the ROS1 era.

- **Blocking Calls (Critical):** It uses `time.sleep(0.1)` inside callback functions. This paralyzes the ROS single-threaded callback system, potentially freezing the robot for 0.5 seconds during operation.
    
- **Legacy Version Checks:** The code includes `if python_version == '2':`. Since ROS2 is strictly Python 3 based, this proves the code is a result of old "copy-paste" practices.
    
- **Global Variable Abuse:** Dozens of unmanaged global variables are scattered throughout the code, making maintenance impossible.
    

#### 1.1.5 Conclusion: Not Production Ready

In conclusion, while the official (or semi-official) code might suffice for a **"demo to check if the sensor works,"** it is absolutely unsuitable for a **"Robotics Product"** where reliability is paramount.

This is precisely why I must reinvent the wheel—or rather, why I must **"forge a proper wheel."**

### 1.2 Issue & PR Analysis

![WIT issue maintenance](/assets/img/2026-01-03-WIT-refactoring/WIT_issue.png)

To gauge the user sentiment towards the repository, I visited the Issue tab.

- **Lack of Maintenance:** While issues are open, almost none have been resolved and closed properly. Most are either closed by the authors themselves out of frustration or simply left abandoned.
    
- **Irrelevant Responses:** In response to a query like "The 50Hz setting doesn't work in ROS2," the replies seem to come from a CS agent simply copy-pasting the manual rather than a developer. This is clear evidence that no meaningful technical communication is taking place.
    
- **PR (Pull Request):** There are no meaningful contributions or merged codes. This indicates that the project is no longer being improved.
    

### 1.3 Forks & Insight

Hoping that perhaps a "hidden master" had forked and fixed the code, I dug into the Insight tab.

- **Result:** There were no starred forks, nor any repositories with meaningful additional commits. It seems truly no one has touched it.
    

### 1.4 GitHub Search Results

I searched GitHub for unofficial codes and found a few drivers written in Python and one C++ project that, while functional, had room for improvement.

- **Python Version:** Excluded due to performance and real-time constraints.
    
- **C++ Version:** There is an existing project that functions well. However, to achieve the core objectives of this project—**implementing ROS2 Lifecycle Nodes and a flexible Component-based structure**—I concluded that designing from scratch is more appropriate than modifying the existing code structure.
    

---

## 2. Strategy Formulation

> **[🔗 WIT official download center](https://www.wit-motion.com/file.html)**

> **[🔗 WIT official google drive](https://drive.google.com/drive/folders/1I6sBC-8Q3_vtY-GrFDZbWJZJFk7UnNfO)**

I have secured the official **protocol documentation** and **datasheets**. The hardware specs are innocent; only the software is guilty.

### 2.1 Target Model Selection

While supporting every model would be ideal, I decided to focus on the three most cost-effective models commonly used in robot development: **WT901C, WT61C, and WT931**.

### 2.2 Development Goals

- **ROS2 Lifecycle Node Application:** Ensuring system stability through a managed state machine.
    
- **Strict Type Checking:** Strengthening data integrity verification for serial communication.
    
- **Zero-Copy:** Minimizing unnecessary memory copying.
    

Now, let's go devour that datasheet.

---

**[Closing: For Those in a Hurry]**

This project is expected to proceed with a long-term view to achieve my personal engineering goals of **mastering Modern C++** and **strictly adhering to ROS2 Lifecycle standards**. (This is the unavoidable reality of a full-time employee developing in spare time after work.)

Therefore, if you need to deploy a robot right now and functional operation (Topic Publish) is your top priority, I recommend checking out the **[existing C++ repository (Ericsii/ros_wit_imu_node)](https://github.com/Ericsii/ros_wit_imu_node)** mentioned earlier.

However, if you are interested in **stable state management (Lifecycle)** and the **struggles of the development process**, I invite you to subscribe to this series and join me in pioneering this 'wild land'.
