---
layout: post
title:  "Maintaining RPLidar Driver: Retrospective on RPLidar ROS 2 Driver v1.0"
date:   2026-01-11 00:00:00 +0900
categories: [Robotics, ROS2]
tags: [C++, Maintenance, Open Source, RPLidar, Middleware]
description: "Releasing into the Wild: From Coder to Maintainer"
---
## 1. The v1.0 Release: The Beginning of the Hard Part

In software engineering, tagging version `1.0.0` does not mean "completion". Instead, it signals the beginning of **"maintenance hell"**—the moment the code leaves my local machine to run in uncontrolled environments.

Once my personal project became a public asset, I had to transition from a simple coder to a "maintainer."

## 2. Promotion: Code Without Users is Dead Code

RPLidar holds a dominant share of the low-cost LiDAR market. However, the existing driver is not enough, failing to keep up with the latest ROS 2 features. The demand was clear.

Even the best code is meaningless if no one uses it. I announced the release on major communities like **ROS Discourse** and **Reddit (r/ROS)**.

## 3. Feedback: Validation, Questions, and Criticism

The response was more active than expected. The feedback fell into three main categories.

### (1) Validation

Many users responded with comments like "Thanks for solving the frustration with the old driver" or "I needed Lifecycle Node support." This confirmed that the "pain points" I identified were real and that my design direction was correct.

### (2) Questions (The "Why")

"Why was this feature designed this way?" "Is this supported?" The users' questions were sharp. To answer them, I had to re-examine my code and clarify my technical rationale. For example, a question regarding the role of **Diagnostics** led me to clearly define the boundary between **Diagnostics** and **Watchdogs**. These questions forced me to grow.

### (3) Criticism

This was the most painful part, but also where I learned the most.

- **Over-promising:** Some pointed out that the `README` felt too aggressive or over-promised. I admit this. My enthusiasm led to strong wording. However, what's done is done. The only solution is not to make excuses, but **to continue developing until the quality matches the strong claims in the README.**
    
- **AI Usage Controversy:** I used Generative AI to refine my English nuances and communication. This seemed to trigger the "anti-AI" sentiment, with some users feeling it was unnatural. However, my conclusion is clear: AI is merely a **tool** for efficiency. Instead of arguing with those who criticize the tool itself, **I simply need to work to the results.**
    

## 4. Insight: The Silent Majority

The biggest insight from this process is the importance of the **"Silent Majority."** Only a small minority of users leave comments on forums. Most users do not care about the internal structure of the driver or the development process. They just want valid sensor data when they type `ros2 launch`.

The majority want to treat the driver as a **"Black Box"**—something that just works. Instead of swaying with every piece of feedback, I learned that a maintainer's true job is to **steadily move toward the next milestone** for those silent users who simply `git clone` and use the software.

## 5. Lessons Learned

At first, No matter how well a single developer or a small team prepares (Close), they cannot match the feedback received after a public release (Open). However, a premature release can lead to technical debt and a loss of trust. Balancing "perfect preparation" and "rapid feedback" is crucial. 
I select premature release and learned three things

- **Lack of Test Procedures:** I do not own every model of RPLidar. I should have asked users to test them, but my request process and report format were poor. Instead of just saying "Please test this," I should have specified **how** to test and **what** logs to provide.
    
- **Issue Templates:** Without templates initially, users were confused about how to format their logs or reports.
    
- **Versioning:** It might have been better to start with `0.9` following Semantic Versioning. Bugs are inevitable, and usage patterns often change frequently in the early stages.


## 6. Future Direction: Scope & Standard

Beyond just making a "better driver," I now have clear goals.

- **Standard (REP):** ROS 2 provides detailed standards (REPs) for hardware interfaces. I want to bring the level of standard compliance, typically found in high-end equipment, to the low-cost RPLidar.
    
- **Architecture:** I aim to establish a solid architecture for this driver so it can serve as a reference for developing other LiDAR drivers in the future.
    

However, I must define the **Scope**. I focus on the ROS 2 interface and system integration. Bugs within the hardware SDK or serial communication issues are the manufacturer's responsibility. I cannot shoulder everything. **I am a robotics engineer, not a firmware engineer.**