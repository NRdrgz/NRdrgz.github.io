---
layout: post
title: "Just1: How I built a small autonomous robot ..."
date: 2025-10-22
thumbnail: /assets/images/posts/thumbnails/just1_thumbnail.jpg
hero_image: /assets/images/posts/hero-images/just1_hero.jpg
excerpt: "How I built Just1, a $250 Mecanum wheel robot that can navigate and avoid obstacles autonomously."
---

Picture this: You're sitting in your room, scrolling through robotics videos on YouTube, watching these incredible autonomous robots navigate through complex environments. The price tags? Thousands of dollars. The complexity? Intimidating. But what if I told you that you could build your own autonomous robot for just $250?


## The Genesis of an Idea

<div class="story-bar">
It all started when I was working for an autonomous driving company in San Francisco. I was surrounded by tremendously smart people working on a very complex system. I already knew about robotics, mainly from the software and data side, but I wanted to better understand several other aspects: middleware, perception, planning, control, and more. Convinced that hands-on projects are the best way to learn and make progress, I started working on Just1. I wanted to build a small, affordable robot that could navigate manually or autonomously and avoid obstacles.
</div>

![Just1 Cartoon](/assets/images/posts/content/just1_cartoon1.png){: .img-medium}

## The Hardware Challenge

Building a robot on a budget means making some tough choices. Every component had to justify its cost, and every feature had to serve a clear purpose.

**The Brain**: Raspberry Pi 4 (8GB) at $80 was my biggest expense, but it needed to handle real-time sensor processing, SLAM algorithms, and navigation planning. The Pi 5 would have been nice, but finding a reliable 5A power supply proved challenging. Looking back, the RAM isn't much of a bottleneck since we're using LIDAR instead of camera-only navigation. We're much more constrained by CPU usage. 

**The Eyes**: This is where things got interesting. I originally planned to use only a camera for Visual SLAM (VSLAM), but after weeks of debugging and optimization, I ended up buying the cheapest 2D Lidar I could find. The LD19 Lidar at $70 became the cornerstone of the robot's perception system.

**The Movement**: Mecanum wheels were fun to work with. I initially wanted to experiment with them because they provide interesting control capabilities. These omnidirectional wheels allow the robot to move forward, backward, sideways, and rotate in place, making them perfect for tight spaces and precise positioning. Four TT motors with L298N drivers were the cheapest solution to control them.

**The Senses**: An MPU6050 IMU ($3!) provides orientation data, while the optional Camera Module 3 adds visual feedback for future enhancements.

<div class="story-bar">
The total cost breakdown: $250 for a fully functional autonomous robot. <br>
That's less than many people spend on a single component for their robotics projects!
</div>

## The Software Stack: Where the Magic Happens

Hardware is just the beginning. The real challenge was making all these components work together seamlessly. I chose ROS2 as the middleware because it's the industry standard for robotics, and learning it would be valuable for future projects. My main goal for this project was actually to get hands-on experience with ROS2 to better understand how it works.

**Navigation Stack**: RTAB-Map handles SLAM (Simultaneous Localization and Mapping), creating a map of the environment while tracking the robot's position. Nav2 provides the path planning and obstacle avoidance capabilities.

**Sensor Fusion**: A Madgwick filter combines IMU data with other sensors for accurate orientation estimation.

**Visualization**: Coming from a company where dozens of engineers work on a single tool to visualize sensor data, I couldn't afford the time and resources to reinvent the wheel.
Foxglove Studio became my best friend during development. This powerful tool lets me visualize sensor data, monitor the robot's state, and even send navigation goals through an intuitive interface. 

## The Development Journey

Building Just1 wasn't just about assembling hardware and writing code. It was a crash course in robotics engineering. Here are some of the key lessons I learned:

### Lesson 1: Start Simple, Then Optimize
My first version used JPEG encoding for the camera feed, which consumed massive CPU resources. Switching to H.264 encoding dramatically improved performance, but I didn't need to spend time on that until late in the development. Sometimes the simplest solution isn't the most efficient, but don't spend time on issues you don't have yet.

### Lesson 2: Documentation is Everything
Every wiring connection, every software configuration, every troubleshooting step. I documented it all. This wasn't just for others who might want to build their own Just1, but for future me who would inevitably forget how I solved that one tricky bug at 2 AM.

### Lesson 3: Real-World Testing Reveals Everything
Simulation is great, but there's no substitute for real-world testing. Coming from the software world into hardware, you have to expect the unexpected. Your robot can fail or behave strangely simply because it's out of battery. Light and shadows can play tricks on your sensor inputs. 

<a href="https://www.youtube.com/watch?v=m_tZ3eg-KsA" target="_blank">
  <img src="/assets/images/posts/content/just1_demo.png" 
       alt="Watch the demo" 
       style="width:1000px;"/>
</a>

## The Foxglove Revelation

One of the most surprising discoveries was how much Foxglove Studio transformed the development experience. This visualization tool became essential for:

- **Real-time monitoring**: Watching sensor data streams, camera feeds, and navigation status
- **Debugging**: Visualizing the robot's understanding of its environment
- **Control**: Sending navigation goals through an intuitive interface
- **Analysis**: Reviewing recorded data to understand what went wrong (or right)

Foxglove turned what could have been a command-line nightmare into an intuitive, visual experience. It's like having a cockpit for your robot. I've written a  {% include link_prominent.html url="https://nrdrgz.github.io/2025/10/21/foxglove-studio-robotics-visualization/" text="dedicated blog post on Foxglove"%}.


## The Bigger Picture

Building Just1 taught me that autonomous robotics doesn't have to be prohibitively expensive or impossibly complex. Having previously worked with the famous {% include link_prominent.html url="https://github.com/TheRobotStudio/SO-ARM100" text="SO-101" %}, I'm proud to have built something similar for a small autonomous robot, complete with ROS2 integration.

The project is fully open-source, with complete documentation, CAD files, and software available for anyone who wants to build their own. I built Just1 mainly for myself, but I was proud to share it online and received great feedback. Just1 has been shared {% include link_prominent.html url="https://www.reddit.com/r/robotics/comments/1noj43w/meet_just1_a_small_mecanumwheel_autonomous_robot/" text="on Reddit" %} and appeared in the {% include link_prominent.html url="https://www.weeklyrobotics.com/weekly-robotics-330" text="Weekly Robotics newsletter" %}! I ended up meeting new people and having drinks with others who wanted to build their own Just1 and learn robotics. That's the beauty of hardware!


## The Takeaway

The best way to learn about something is just to dive into it and build a project out of it. In the software world, it's very easy to do. You just need a computer and you can start coding. But with Just1, I realized that the same is also possible in hardware. You can build a very simple robot and learn principles that are applied in robotaxis roaming the streets of San Francisco. 

The journey from idea to working robot was filled with challenges, late-night debugging sessions, and moments of pure joy when everything finally clicked. But most importantly, it showed me that the barrier to entry in robotics isn't as high as it might seem.

So, what's your "Just1" project? What have you been wanting to build but thought was too complex or expensive? Sometimes, the best way to find out is to just start building.

<div class="callout">
Want to build your own Just1? Check out the complete project documentation, CAD files, and software at the {% include link_prominent.html url="https://github.com/NRdrgz/Just1" text="Just1 GitHub repository" %}. All the code, schematics, and build instructions are freely available for anyone who wants to dive into autonomous robotics.
</div>


