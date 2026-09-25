# Whitewolf7658

**Contact:** Hwnwx/1257601320635076609 | https://www.roblox.com/users/1660464640/profile

*Focus Areas: 3D Coordinate Spaces, Custom Physics, Script Optimization, Mathematical Systems, Physics.*

*Core Strengths: Debugging complex systems, tracking down memory leaks and performance issues, optimizing existing code, and building scalable systems and frameworks.*


---

## About Me!

When I first tried making my own game back in early 2023, I was pretty quickly introduced into just how complicated Roblox Studio looked and how confusing scripting was at the time. I was completely clueless about basically everything, from using Studio properly to understanding how different systems work, how they were built and how they communicated eventually with each other.

Instead of giving up, I tried focusing on actually learning how programming/scripting worked. I decided I was going to try dedicate some time each day to learning, in turn I ended up spending countless days learning the basics and slowly started making progress, but eventually reached a point where I frankly didn't know what I was supposed to learn next, I was at a complete stalemate. After some time I decided I'd try reaching out to some friends to see if they had anything to say, and one of them ended up introducing me to a small group of experienced programmers and computer science students in university who became my mentors. They helped me understand the more difficult mathematical and backend concepts that I was struggling to learn by myself.

For almost the last three years, I've tried to dedicate around 2–3 hours a day to improving my scripting. Most of my learning follows the same process: learn a concept, try implementing it myself in Luau, find what I've done wrong, debug it, and then apply it to an actual Roblox system, Then try again. This is how I've worked through things like rotated 3D spaces, object intersections, custom physics and momentum.

The two systems below are the result of that process. With the exception of the Drone and Ring models/assets, all of the scripting is my own. My mentors were a massive help in teaching me, but the actual implementation and debugging was done entirely by me. Both projects took roughly 3–4 months of development, with a lot of that time spent dealing with physics edge cases, networking, memory leaks and the performance problems.

*I also created the SFX for both the Portal and Drone.*

## Project 1: The Quantum Portal System (Taken some inspiration from the Dr Strange Portal, hence the ring)
This is a working portal where you can actually see the other side! I built it from scratch to handle things like 3D physics momentum, player tracking across frames, and smooth rendering illusions between things called unaligned spaces. 

### Showcase & Mechanics

> 🔗 **[Click Here to Watch the Portal Showcase](https://medal.tv/games/screen-capture/clips/nAWRFFYqugyyWB-mL?invite=cr-MSwyZzUsNDI3MTQwMjg3)**
> Note: Clip Runs smoother on website

---

### Portal Development Logs & Solved Bottlenecks
This system underwent multiple revisions to identify and eliminate edge cases, visual artifacts, and CPU performance drops. Below are the core technical problems solved during development:

* **Virtual Camera Obstruction:** Moving the viewport camera backwards improved the depth of the portal view as I had intended it to do, but it also caused the wall behind the portal in which I was exiting out of to become visible. After some testing I fixed this by hiding the host wall only within the ViewportFrame.
* **Perspective Distortion:** I originally tried using a much wider Field of View (100°+) to make it seem like the character was actually infront of the portal, but this heavily distorted the portal view. I ended up using a fixed camera distance of 8 studs with a 52° Field of View instead, I tried it again and achieved the look I was wanting.
* **Character Clipping:** Once the backing wall was hidden, characters behind the camera used for the portal could sometimes appear inside the portal view. I fixed this by checking their local Z position relative to the portal and then excluding them when they were behind the camera.
* **Threshold Flickering:** While experimenting I used exactly `Z = 0` to switch rendering states, this caused flickering when the player's position moved slightly between frames. After multiple failed attempts on fixing it, I separated the thresholds to `-0.15` and `0.15` studs so small position changes wouldn't constantly switch the state.
* **Portal Re-Entry:** Teleporting directly onto the destination portal's plane could sometimes immediately trigger the opposite portal and send the player back through. To fix this I added a traversal state and a small safety distance before another crossing could be registered properlly.
* **Portal Wall Collision:** The wall containing the portal could still physically block the player even though the portal itself was visually open. I used the player's local position around the portal opening to temporarily disable the wall collision while they were passing through.
* **RenderStepped Performance:** Updating cloned world geometry every frame became unnecessarily expensive. Static objects are now left alone after being created, while objects that actually need updates are processed separately at 30Hz.
* **Workspace Scanning:** Earlier versions repeatedly searched through large sections of the workspace to find parts that needed updating. I replaced this with cached references between the original parts and their viewport clones, removing the need to constantly search for them again.
* **Terrain Blockiness:** When I attempted to implement the ability to view terrain in the Portal, I tried to create a terrain extraction system. My first attempted produced noticeably stepped terrain around curved surfaces. I stepped in and reduced this by using a 4-stud surface resolution and only generated the visible outer terrain instead of unnecessary interior voxels.


---

###  Portal System Code
Click any of the links below to view the core scripts utilized.

* 📄 **[View Source: Math & Coordinate Matrix Library (PortalMath)](./scripts/portal-math.md)**
* 📄 **[View Source: Player Teleportation & Camera Handoff Client](./scripts/portal-traversal.md)**
* 📄 **[View Source: Procedural Ring Particle & Lighting Effects](./scripts/portal-vfx.md)**
* 📄 **[View Source: Viewport World Streaming & Terrain Extraction Client](./scripts/portal-render.md)**

---

## Project 2: Security Drone (Drone System)
I created an accurate, physics based drone platform built from scratch. The system controls manual rigid body kinematics, a real time ground tracking feature, and strict server safety features to ensure the drone runs smoothly in active servers.

### Showcase & Mechanics
> 🔗 **[Click Here to Watch the Security Drone Flight Showcase](https://medal.tv/games/screen-capture/clips/nAWJQR6hVfKJZhuxB?invite=cr-MSw1eVEsNDI3MTQwMjg3)**
> Note: Clip Runs smoother on website

---

### Drone Development Logs & Solved Bottlenecks

* **Duplicate Flashlight Input:** The floodlight was originally being controlled by both the flight and camera scripts, this in turn caused an inconsistent toggling when F was pressed. To fix this I removed the duplicate input handling and made the camera client the only script responsible for the flashlight's input.
* **Flashlight Response Delay:** The floodlight originally waited for the server state to replicate before turning on, this resulted in the light feeling slightly delayed. After seeing what might work, I added local prediction so the light turns on immediately. All while the server still validates and controls the actual state.
* **Packed Drone Controls:** Because the same drone instance is stored in ReplicatedStorage when it is in the players inventory, the camera and flight scripts could still connect to it and respond to the inputs. I added deployment checks so the camera, flight and all accessory controls only work while the drone is in Workspace.
* **Server-Side Command Validation:** Disabling the controls on the client wasn't enough, since remote events could still be fired while the drone was in the users inventory. I added the same deployment checks on the server so flight, look and command inputs are ignored unless the drone has run its deployment script and is inside the Workspace.
* **Drone State Persistence:** After picking up the drone, the drone could leave values such as: Pilot ownership, precision mode, floodlight state and a laser state active. To combat this I added a full state reset when the drone is in the inventory so none of these values are to carry over into the next deployment.
* **Deployer Inventory Duplication:** After placing the drone, the deployer could sometimes remain in the player's inventory because the removal system had relied on a specific Tool name. I changed it to identify the deployer using both its name and the placement script, then remove the correct Tool after it had been placed.
* **Deployer Restoration:** Storing the drone originally tried to restore a hard coded StarterPack.DroneDeployer, which would fail if the actual Tool had a different name. I changed the system to cache the real deployer Tool and restore that version when the drone is in the users inventory.
* **Redeployment Position Mismatch:** The placement preview and server originally calculated the drone's final position separately, which could make the deployed position slightly different from the preview. After checking the script, I changed it so the exact preview transform is sent to the server, with the server applying the same ground offset when placing the drone.
* **Residual Movement After Redeployment:** Because the packed drone reuses the same physical model, it could keep small amounts of linear or angular velocity from its previous flight. To fix this, I had to clear its assembly velocities before and after redeployment so it can't drift or jump away from where it was placed.
* **Old Position Flash:** While testing I discovered moving a stored drone back into Workspace could briefly show it at its previous position before it was moved to the new one. I fixed this by positioning the drone before parenting it back into Workspace, then applying the final transform again afterwards.
* **Placement Rotation Drift:** In the earlier versions of the drone, It recalculated the preview rotation from the player's character direction every frame, this meant that simply turning the character would rotate the drone's placement preview. I changed it so the starting direction is captured once when the deployer is equipped, and rotation only changes when `R` is pressed.
* **Camera Script Failure:** One version of the camera client became corrupted and stopped the drone camera system from working entirely. I restored the last working version and made the later fixes separately instead of changing several parts of the camera system at once.
* **HUD Overlap:** Parts of the drone camera HUD overlapped Roblox's built in menu and hotbar. I fixed this by changing those UI elements individually instead of changing the GUI inset behaviour for the entire interface.
* **Missing Flight Remotes:** During one of my tests, the deployment script accidentally replaced the actual flight server, meaning that the remotes the drone was using such as FlightInput and DroneLoo` were no longer being created. Once I discovered this bug I separated the deployment and flight scripts again so they each handle their own part of the system.

---

### Drone System Code
Click any of the links below to view the core scripts utilized:

* 📄 **[View Source: Drone Flight Server Framework (DroneFlightServer)](./scripts/drone-server.md)**
* 📄 **[View Source: Drone Flight Controller Physics Module (FlightController)](./scripts/drone-physics.md)**

---

## Previous Roblox Development Experience

In addition to the projects demonstrated in this portfolio, I have previously contributed to other Roblox experiences that have reached quite large player audiences.


#### 🚜 Survive Anton Chigurh's Tractor

**~7.4M visits**
https://www.roblox.com/games/124083111510656/Survive-Anton-Chigurhs-Tractor

I contributed to the scripting and maintenance of the game. Most of my time on the project was spent debugging existing systems, tracking down what was causing gameplay issues, fixing broken or inconsistent behaviour, and working on scripts as the game continued development.

I also helped troubleshoot problems that came up during testing and after changes were pushed to the game.

#### Blood Debt – Gun System

**~2.4M visits**
https://www.roblox.com/games/130490210702949/Blood-Debt-Gun-System

I was responsible for the large majority of the scripting behind the gun system. I worked on the core weapon functionality and gameplay behaviour, as well as debugging and maintaining the system as it developed.

A significant amount of the gun system's code was written by me, with a lot of my later work going towards tracking down bugs and fixing issues with the weapon mechanics.

> **Note:** I am no longer associated with either projects nor do I have access to the Dev Studios, project files, or screenshots from either of these projects, mainly because I signed an NDA for Anton and I have deleted everything I had from Blood Debt Gun System from my PC. Because of this, I've listed them as previous experience rather than full portfolio showcases. The projects shown earlier in this portfolio are work that I can directly provide.
