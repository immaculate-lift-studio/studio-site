+++
date = '2026-01-25T20:28:59-08:00'
draft = false
title = 'Smoke and Mirrors: Making an Open World Busy'
+++

One of my favorite aspects of game development is that 99% of the time, what people see isn't what they think: it's all smoke and mirrors. Whether it's a person that doesn't "switch on" until you are right next to them or a tree that isn't really a tree until you crash into it, it boils down to deciding what your players can afford to notice in order for the game to appear smooth and enjoyable for the players. The Godot engine doesn't include any easy solutions for managing large worlds, so I wanted to talk about the techniques that made this world possible for me. This is all very high level and I don't go into a lot of detail, but if anyone wants to know more specific techniques or see the code, please contact me.

When dealing with game logic in a large world, there are 2 main elements I focused on: The static, unmoving world and the dynamic, constantly changing world. Both can be incredibly demanding (even when assets are optimized) and because of the conceptual differences between the two, they need very different approaches to fix. For the static world, I made a grid system and optimized draw calls, and for the dynamic world, I made a dynamic bubble of activation. I will explain both below. 

## Managing the Static World
In a world that is 10 or more kilometers wide, you cannot have everything loaded at once. Maybe on more expensive hardware you could hold it in memory, but the calculations that are required would quickly add up and rendering even a static scene would be very difficult to do smoothly. 

The first issue is draw calls - for those unfamiliar with this term, a draw call what a computer's GPU does when it draws something on screen. Each unique object or material generally requires its own draw call. So what do you think happens when you put 4500 buildings, hundreds of streets pieces, and 50,000 trees in a world? It's awful and will probably set your computer on fire, even if the player isn't moving. The key to fix this is **instanced rendering**. Identical objects can be drawn many times with a single draw call, even if the objects are sized or placed differently, and this also applies to a texture on said mesh. This means that when the camera focuses on a city block containing several different types of buildings and street pieces totaling ~400 draw calls with textures, we are now only producing ~12 draw calls. That's a massive improvement, and it gets even better when you focus on the trees: I have thousands of trees rendered, but they are all based on 1 base tree mesh, scaled and rotated randomly, and then placed (in chunked areas) according to a perlin noise algorithm. Instead of thousands and thousands of draw calls, you might only have to process 1-10 on your screen!

![Draw Calls Example](../draw_calls.png)  
*A visual of the chunking system, where each square (roughly) represents 1 draw call for that group of trees*  


The second issue I encountered was collisions, which even for static objects can seriously hamper performance. Any object that can collide with another object (in this game, trees and buildings must have collision) has to report it's position and scan for other objects interacting with it every time the physics server ticks (the average for a game engine is usually 60 times a second). We lighten the load on the hardware by dividing up the world into "grid squares" that are 1km by 1km wide. Within each square, tasks such as physics activation and tracking collision are disabled until the player crosses into that specific grid - which can reduce the computational load of the physics engine tracking colliders by a huge margin. When the player exits that activated grid squared, everything shuts off and sleeps until it's reactivated. Since all of it is static, there's no need for anything more complicated than an on/off switch.

## The Bubble System: Managing the Dynamic World
Things get more complicated when you take a look at the dynamic elements in this game, though. The game features little people that walk around, interact with you if you get near, run away from the helicopter, etc. I also added vehicles that follow traffic laws, avoid each other/the people, and react excessively when a helicopter gets close. But if you add up how many pedestrians and cars were being rendered and calculated (ie how many physics bodies were being moved around and interacting with the world), we had over 3000 citizens and around 1500 cars all operating whether the player was close enough to notice or not. Obviously that's not efficient, but I couldn't use the grid system for this as these are dynamic parts of the world. If I ran the simulations in a small controlled environment, I couldn't get past a couple hundred of either without serious performance issues. So I created a "bubble" system that follows the player and controls when NPCs spawn, how they behave, and how much detail they are given. This system is shaped like a sphere (with the limits roughly correlating to how far the player can see out into the world), and inside the bubble, NPCs can exist are simulated in various detail levels based on proximity to the player. Outside the bubble, they do not exist. As the player moves around the world, NPCs will spawn at randomized (yet controlled) locations within the bubble, and as they move outside the bubble, they will be removed and saved to be reused in another random placement later.

![Bubble System](../bubble_system.png)  
*Clumsy approximation of the bubble system in action*  

The advantages of this system are not only much better performance due to the tiny computational cost as compared to simulating and controlling the entire world at once, but also the opportunity to fill that bubble much more densely than if I had populated the whole world. As the player flies and moves around the world, they experience a fully populated environment that is consistently bustling and interesting.

There is another layer of this to improve efficiency as well: Specific distance-based level of detail (LOD) for each NPC. For citizens, this could mean that one 10 meters away is fully simulated, with animations, reactions, and details, but a citizen 200 meters away is a simple 2d picture that slides across the ground. For vehicles, they don't actually notice or react to other cars and obstacles when the player is hundreds of meters away, they just follow a simple path on the road and when activated, they will start behaving more realistically. So if the player is flying around the world at the high detail level, they will notice and appreciate maybe 200 active cars/citizens, but only 20-30 of them will be fully simulated.

## Combining the two systems
Combining these two systems leads to a perceptually seamless and populated world that maintains performance, only compromising in ways that the player will not notice 99.9% of the time, if ever. My journey to figure all this out required me to shift my understanding of simulation from "how can I do this realistically" to "what does the player need to **percieve** in order to be fooled?" Coming to the conclusion that good enough is far better than perfect as I develop has helped me immeasurably, as I've lost a lot of time to over-analyzing, over-profiling, and over-focusing on tiny things that the player will never see. 

I think that developers going on the same journey as me should not look for a single, one-size-fits-all tool, because different kinds of content have different optimization requirements. My technique of Static (Spatial Organization) and Dynamic (Distance-based) has paid off in spades. The result in RotorSim 2 is:
* Almost 5000 buildings 
* Dozens of kilometers of roads
* Over 50k trees 
* Hundreds of citizens/vehicles behaving realistically wherever the player goes

This all happens with with comparable performance to my first game (which was a fraction of the size and didn't simulate traffic). That means 60-90 frames/second with low end and integrated hardware.

*A disclaimer = PROFILE, PROFILE, PROFILE! I would not have figured any of this out if I hadn't used the built in performance profiler and looked deep within it. Once you figure out what the true cost of a frame is, then you can take steps to optimize it. Without understanding your frames, you won't get anywhere with this kind of optimization.*

