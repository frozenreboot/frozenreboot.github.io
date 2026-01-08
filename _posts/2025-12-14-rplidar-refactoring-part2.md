---
layout: post
title:  "Refactoring RPLidar S3 Driver: From Vendor SDK to Modern ROS2 Node (Part 2)"
subtitle: "Architectural analysis: Bridging the gap between legacy SDKs and production requirements"
date:   2025-12-14 14:00:00 +0900
background: '/assets/img/2025-12-14-rplidar-refactoring/architecture_blueprint.jpg'
categories: [Robotics, ROS2, Architecture]
tags: [C++, Refactoring, Design Patterns, Lifecycle, FSM]
description: "Why I kept the legacy SDK but rewrote the driver. An analysis of anti-patterns like the 'God Object' and 'Thread Starvation', and a blueprint for a self-healing architecture."
---

## Introduction

In part 1, We find some problems in legacy driver. Instead of just patching the existing legacy structure, I decided to **overhaul** the architecture specifically for ROS2. The plan is simple: Delete the old logic and structure.only keep the essentials to be a **True ROS2-native driver**

---

## 1. Structural Analysis: Dissecting the Bloat

A quick `tree` command reveals the reality of the project structure. As expected from a vendor-provided package designed for broad compatibility, it suffers from significant "Platform Bloat."

```text
.
├── launch/             # Deployment configurations
├── sdk/                # Vendor SDK (The Legacy Core)
│   └── src
│       └── arch
│           ├── linux   # Target Platform
│           ├── macOS   # Irrelevant (Bloat)
│           └── win32   # Irrelevant (Bloat)
├── src/                # ROS2 Node Logic (Tight Coupling)
└── package.xml
````

#### The Problem: Platform Noise

The `sdk` directory contains mixed implementations for `win32`, `macOS`, and `linux`. While this approach maximizes compatibility for the vendor, for us—building a robot running on Embedded Linux—it acts as **noise**. It increases build complexity and reduces code readability without adding any value to our specific runtime environment.

-----

## 2. Strategic Decision: Why Keep the SDK?

This raises a fundamental engineering question:
*"If the SDK structure is generic and bloated, why not rewrite the communication layer (`read/write`) from scratch?"*

I decided to **retain the vendor's SDK but wrap it carefully**. Here is the rationale behind this engineering decision:

#### A. ROI (Return on Investment)

Engineering is about efficient resource allocation. Rewriting the serial protocol stack byte-by-byte is essentially **"Reinventing the wheel."** The existing SDK, despite its structural flaws, has a proven communication stack. It is more economical to reuse the working logic than to build a new one from the ground up.

#### B. Maintainability & Forward Compatibility

Hardware vendors frequently update their SDKs to support new firmware or new models (e.g., S2 to S3). If we were to write our own communication layer, we would incur **Technical Debt**, forcing us to manually update our code every time the vendor releases a new feature.

**The Solution: The Wrapper Pattern**
Instead of modifying or discarding the SDK, I treat it as a **Black Box**. By encapsulating the dirty SDK internals within a clean C++ Wrapper Class, we can isolate the dependencies. This allows us to focus on what really matters: **ROS2 Lifecycle integration** and **System Stability**.

-----

## 3\. AS-IS Analysis: Identifying Anti-Patterns

With the structure defined, let's look at the core logic (`rplidar_node.cpp`). I identified three critical architectural flaws that compromise system stability.

### 3-1. The "God Object" (Violation of SRP)

The current node file does too much. It handles:

  * **Networking:** Low-level serial port management.
  * **Parsing:** Decoding proprietary binary protocols.
  * **Application:** Managing ROS2 publishers and parameters.

This is a clear violation of the **Single Responsibility Principle (SRP)**. The tight coupling makes unit testing impossible and maintenance a nightmare.

### 3-2. Thread Starvation (CRITICAL)

This is the most severe issue. The legacy driver uses a **Blocking I/O** model, polling hardware data inside the main thread's loop.

Since ROS2's default Executor is single-threaded, while the driver is waiting for sensor data, it cannot process other events. This leads to **Thread Starvation**, where critical callbacks—such as Parameter updates, Heartbeats, or Lifecycle transitions—are delayed or ignored. Effectively, the node becomes **"unresponsive"** to the rest of the system while reading data.

### 3-3. Manual Memory Management

The codebase is littered with manual `new` and `delete` calls. In the era of Modern C++ (C++14/17), this is an obsolete practice that increases the risk of **Memory Leaks**, especially during exception handling.

-----

## 4 TO-BE Architecture: The Blueprint

Based on this analysis, I designed a new architecture focused on **Fault Tolerance** and **Determinism**.

1.  **Lifecycle Management:**
    Implementing `rclcpp_lifecycle::LifecycleNode` to strictly manage states (`Unconfigured` → `Inactive` → `Active`). This prevents the "spin-on-boot" behavior and allows the system manager to control the driver.

2.  **The "Zombie" FSM (Self-Healing):**
    Decoupling the data acquisition loop from the main ROS2 thread. A background thread runs a **Finite State Machine (FSM)** that detects connection drops and automatically attempts to "resurrect" the driver without crashing the node.

3.  **Modern C++ Standards:**

      * **RAII:** Using `std::unique_ptr` for automatic resource management.
      * **Zero-Copy:** Using `std::move` to publish scan data efficiently.

-----

## 5 Implementation Preview

The implementation of this architecture—the **MVP (Minimum Viable Product)**—is now complete. The legacy dependencies are isolated, and the node is robust against hardware failures.

You can check the full source code in the repository below:

> **[🔗 GitHub Repository: Refactored RPLidar ROS2 Driver](https://github.com/frozenreboot/rplidar_ros2_driver)**

However, writing the code is one thing; understanding *how* it survives hardware disconnections is another.
In **Part 3**, I will dive deep into the implementation details of the **"Zombie Loop"**, explain how to achieve **Non-blocking I/O** in ROS2, and share the results of the **Hz Performance Test**.

*(To be continued...)*
