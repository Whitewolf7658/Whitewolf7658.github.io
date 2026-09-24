# Whitewolf7658
**Contact:** Hwnwx/1257601320635076609 | https://www.roblox.com/users/1660464640/profile
*Focus Areas: 3D Coordinate Space, Custom Physics, and Script Optimization.*
*Core Strengths: Finding hidden memory leaks, cleaning up unoptimized code, and solving general and complex bugs that break game servers and more, Creating script and writing frameworks that can be scaleable.*

---

## 🛠️ About Me
When I first set out to make my very first game back in early 2023, I was quickly rudely awoken to the complex or what I thought at the time complex systems of Roblox Studio and scripting in general! I was completely clueless at the time and confused about just how to use Studio's tools, how to make systems, and how to make them communicate properly. 

Instead of quitting, I decided to focus entirely on learning how programming actually works. I spent countless hours trying to drill the basic concepts in, slowly I saw progress, and as I started moving onto the more advanced side of development. During this, I had told one of my close friends about what I had been doing and my interest in making my own game one day, eventually my friend introduced me to a group of his friends who are currently in university for computer science and are highly experienced in game scripting as they do it themselves, some of them have even made both Indie games on steam and games on Roblox! They essentially became my tutors while I tried to learn, they helped by guiding me through the complex mathematical and the backend design concepts of scripting. They were a crucial part in me being where I am today.

For almost the last three years, I have dedicated 2 to 3 hours daily (Or tried to some days) to truly understanding scripting with the help of what I see as my mentors. My routine had relied heavily on a constant loop, the first step was learning a concept that was hard for me at the time with the help of that group (Things like mapping rotated 3D spaces, calculating an exact object intersections, or handling custom physics momentum) and then consolidating my knowledge through writing a Luau code to verify my learning, I would then figure out the errors and how to fix them, and finally try translating that knowledge directly over to Roblox with the help of the group my friend introduced me to.

The two systems below were built completely from scratch, with the exception of the Drone and Ring models/assets, as I don't specialize in 3D modelling (All of the scripting is entirely my own) While my mentors were a massive help in teaching me throughout the process, the actual development, debugging, and building of these systems was done by myself. Both took roughly 3–4 months of fairly intense daily coding and a lot of trial and error, especially when tracking down physics edge cases, memory leaks, and issues with server remotes. A lot of the work also went into making sure the systems weren't unnecessarily expensive to run. This included keeping networking under control, validating important actions on the server, cleaning up connections and instances properly, and making sure everything remained stable with multiple players using the systems at once.

## Project 1: The Quantum Portal System (Taken some inspiration from the Dr Strange Portal. Hence the ring)
This is a working portal where you can actually see the other side! I built it from scratch to handle things like 3D physics momentum, player tracking across frames, and smooth rendering illusions between things called unaligned spaces.

### Showcase & Mechanics

> 🔗 **[Click Here to Watch the Portal Showcase](https://medal.tv/games/roblox/clips/nAEdFZa0b5uy4qNvv?invite=cr-MSxnS3ksNDI3MTQwMjg3)**
---

### Portal Development Logs & Solved Bottlenecks
This system underwent multiple revisions to identify and eliminate edge cases, visual artifacts, and CPU performance drops. Below are the core technical problems solved during development:

* **Virtual Camera Obstruction:** Moving the viewport camera backwards improved the depth of the portal view, but also caused the wall behind the exit portal to become visible. I fixed this by hiding the host wall only within the ViewportFrame.
* **Perspective Distortion:** I originally tried using a much wider Field of View (100°+), but this heavily distorted the portal view. I ended up using a fixed camera distance of 8 studs with a 52° Field of View instead.
* **Character Clipping:** Once the backing wall was hidden, characters behind the virtual camera could sometimes appear inside the portal view. I fixed this by checking their local Z position relative to the portal and excluding them when they were behind the camera.
* **Threshold Flickering:** Using exactly `Z = 0` to switch rendering states caused flickering when the player's position moved slightly between frames. I separated the thresholds to `-0.15` and `0.15` studs so small position changes wouldn't constantly switch the state.
* **Portal Re-Entry:** Teleporting directly onto the destination portal's plane could immediately trigger the opposite portal and send the player back through. I added a traversal state and a small safety distance before another crossing could be registered.
* **Portal Wall Collision:** The wall containing the portal could still physically block the player even though the portal itself was visually open. I used the player's local position around the portal opening to temporarily disable the wall collision while they were passing through.
* **RenderStepped Performance:** Updating cloned world geometry every frame became unnecessarily expensive. Static objects are now left alone after being created, while objects that actually need updates are processed separately at 30Hz.
* **Workspace Scanning:** Earlier versions repeatedly searched through large sections of the workspace to find parts that needed updating. I replaced this with cached references between the original parts and their viewport clones, removing the need to constantly search for them again.
* **Terrain Blockiness:** My first terrain extraction system produced noticeably stepped terrain around curved surfaces. I reduced this by using a 4-stud surface resolution and only generating the visible outer terrain instead of unnecessary interior voxels.


---

###  Portal System Source Code
Click any of the links below to view the full, documented source files on their own dedicated subpages:

* 📄 **[View Source: Math & Coordinate Matrix Library (PortalMath)](./scripts/portal-math.md)**
* 📄 **[View Source: Player Teleportation & Camera Handoff Client](./scripts/portal-traversal.md)**
* 📄 **[View Source: Procedural Ring Particle & Lighting Effects](./scripts/portal-vfx.md)**
* 📄 **[View Source: Viewport World Streaming & Terrain Extraction Client](./scripts/portal-render.md)**

---

## Project 2: Optimized Security UAV (Drone System)
This is a high-fidelity, physics-based flight simulation platform built from scratch. It handles manual rigid-body kinematics, real-time ground tracking, and strict server safety features to run smoothly in highly populated servers.

### Showcase & Mechanics
> 🔗 **[Click Here to Watch the Security Drone Flight Showcase](https://medal.tv/games/roblox/clips/nAEjE85ifakJekGnp?invite=cr-MSw2bXIsNDI3MTQwMjg3)**

---

### Drone System Source Code
Click any of the links below to view the full, documented source files on their own dedicated subpages:

* 📄 **[View Source: Drone Flight Server Framework (DroneFlightServer)](./scripts/drone-server.md)**
* 📄 **[View Source: Drone Flight Controller Physics Module (FlightController)](./scripts/drone-physics.md)**

---
