# Whitewolf7658

**Contact:** Hwnwx/1257601320635076609 | https://www.roblox.com/users/1660464640/profile

*Focus Areas: 3D Coordinate Space, Custom Physics, and Script Optimization.*

*Core Strengths: Finding hidden memory leaks, cleaning up unoptimized code, and solving general and complex bugs that break game servers and more, Creating script and writing frameworks that can be scaleable.*

---

## About Me!
When I first set out to make my very first game back in early 2023, I was quickly rudely awoken to the complex or what I thought at the time complex systems of Roblox Studio and scripting in general! I was completely clueless at the time and confused about pretty much everything! like how to use Studio's tools, how to make systems, and how to make them communicate properly. 

Instead of quitting, I decided to focus entirely on learning how programming actually works. I spent countless hours trying to drill the basic concepts in, slowly I saw progress, and as I slowly got better I tried moving onto the more advanced side of development but I was struggling to learn as I frankly didn't know what to learn. I had then reached a stalemate.. I didn't know how to progress, or where to look, Eventually I reached out to close friends about what I had been doing and my interest in making my own game one day, my friend then introduced me to a group of his friends older friends who are currently in university for computer science and are highly experienced in game scripting as they do it themselves, some of them have even made both Indie games on steam and games on Roblox! They essentially became my tutors while I tried to learn, they helped by guiding me through the complex mathematical and the backend design concepts of scripting. They were a crucial part in me being where I am today.

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

### Drone Development Logs & Solved Bottlenecks

* **Duplicate Flashlight Input:** The floodlight was originally being controlled by both the flight and camera scripts, which caused inconsistent toggling when `F` was pressed. I removed the duplicate input handling and made the camera client the only script responsible for flashlight input.
* **Flashlight Response Delay:** The floodlight originally waited for the server state to replicate before visually turning on, which made it feel slightly delayed. I added local prediction so the light turns on immediately, while the server still validates and controls its actual state.
* **Packed Drone Controls:** Because the same drone instance is stored in `ReplicatedStorage` when packed, the camera and flight scripts could still hold references to it and respond to inputs. I added deployment checks so the camera, flight and accessory controls only work while the drone is deployed in `Workspace`.
* **Server-Side Command Validation:** Disabling the controls on the client wasn't enough, since remote events could still be fired while the drone was packed. I added the same deployment checks on the server so flight, look and command inputs are ignored unless the drone is currently deployed.
* **Drone State Persistence:** Packing the drone could leave values such as pilot ownership, precision mode, floodlight state and laser state active. I added a full state reset when the drone is packed so none of these values carry over into the next deployment.
* **Deployer Inventory Duplication:** After placing the drone, the deployer could sometimes remain in the player's inventory because the removal system relied on a specific Tool name. I changed it to identify the deployer using both its name and placement script, then remove the correct Tool after deployment.
* **Deployer Restoration:** Packing the drone originally tried to restore a hard-coded `StarterPack.DroneDeployer`, which would fail if the actual Tool had a different name. I changed the system to cache the real deployer Tool and restore that version when the drone is packed.
* **Redeployment Position Mismatch:** The placement preview and server originally calculated the drone's final position separately, which could make the deployed position slightly different from the preview. I changed it so the exact preview transform is sent to the server, with the server applying the same ground offset when placing the drone.
* **Residual Movement After Redeployment:** Because the packed drone reuses the same physical model, it could keep small amounts of linear or angular velocity from its previous flight. I now clear its assembly velocities before and after redeployment so it can't drift or jump away from where it was placed.
* **Old Position Flash:** Moving a stored drone back into `Workspace` could briefly show it at its previous position before it was moved to the new one. I fixed this by positioning the drone before parenting it back into `Workspace`, then applying the final transform again afterwards.
* **Placement Rotation Drift:** Earlier versions recalculated the preview rotation from the player's character direction every frame, meaning that simply turning the character would rotate the placement preview. I changed it so the starting direction is captured once when the deployer is equipped, and rotation only changes when `R` is pressed.
* **Camera Script Failure:** One version of the camera client became corrupted and stopped the drone camera system from working entirely. I restored the last working version and made the later fixes separately instead of changing several parts of the camera system at once.
* **HUD Overlap:** Parts of the drone camera HUD overlapped Roblox's built-in menu and hotbar. I adjusted those UI elements individually instead of changing the GUI inset behaviour for the entire interface.
* **Missing Flight Remotes:** During one revision, the deployment script accidentally replaced the actual flight server, meaning remotes such as `FlightInput` and `DroneLook` were no longer being created. I separated the deployment and flight scripts again so they each handle their own part of the system.

---

### Drone System Source Code
Click any of the links below to view the full, documented source files on their own dedicated subpages:

* 📄 **[View Source: Drone Flight Server Framework (DroneFlightServer)](./scripts/drone-server.md)**
* 📄 **[View Source: Drone Flight Controller Physics Module (FlightController)](./scripts/drone-physics.md)**

---

## Previous Roblox Development Experience

In addition to the projects demonstrated in this portfolio, I have previously contributed to other Roblox experiences that have reached substantial player audiences.


#### 🚜 Survive Anton Chigurh's Tractor

**~7.4M visits**
https://www.roblox.com/games/124083111510656/Survive-Anton-Chigurhs-Tractor

I contributed heavily to the scripting and maintenance of the game. Most of my time on the project was spent debugging existing systems, tracking down what was causing gameplay issues, fixing broken or inconsistent behaviour, and working on scripts as the game continued development.

I also helped troubleshoot problems that came up during testing and after changes were pushed to the game.

#### Blood Debt – Gun System

**~2.4M visits**
https://www.roblox.com/games/130490210702949/Blood-Debt-Gun-System

I was responsible for the large majority of the scripting behind the gun system. I worked on the core weapon functionality and gameplay behaviour, as well as debugging and maintaining the system as it developed.

A significant amount of the gun system's code was written by me, with a lot of my later work going towards tracking down bugs and fixing issues with the weapon mechanics.

> **Note:** I no longer have access to the Dev Studios, project files, or screenshots from either of these projects, mainly because I signed a NDA for Anton and I have deleted all data from Blood Debt Gun System from my PC. Because of this, I've listed them as previous experience rather than full portfolio showcases. The projects shown earlier in this portfolio are work that I can directly provide.
