# Whitewolf7658

**Roblox / Luau Programmer**

**Contact:** Hwnwx / 1257601320635076609  
**Roblox:** [Profile](https://www.roblox.com/users/1660464640/profile)

I’m a Roblox/Luau programmer with around three years of experience, primarily focused on technical gameplay systems, custom physics, 3D/CFrame mathematics, networking, optimization and debugging.

My recent work includes a custom portal system capable of transforming players and momentum between arbitrarily oriented spaces, and a physics-based security drone with custom flight behaviour, client/server networking, deployment, cameras and failsafe systems.

## Portfolio Overview

- [Featured Projects](#featured-projects)
  - [Quantum Portal System](#quantum-portal-system)
  - [Security Drone System](#security-drone-system)
- [Portal Development & Problems Solved](#portal-development--problems-solved)
- [Drone Development & Problems Solved](#drone-development--problems-solved)
- [Previous Roblox Development Experience](#previous-roblox-development-experience)
- [Technical Notes & Design Decisions](#technical-notes--design-decisions)

## Featured Projects

### Quantum Portal System
A custom portal system built around coordinate space transformations, real time portal rendering and momentum preserving traversal.

**Highlights:**
- Arbitrarily oriented portal transformations
- Linear and angular momentum transformation
- Portal crossing and re-entry handling
- Real time ViewportFrame world rendering
- Camera transformation between portal spaces
- Character and terrain rendering
- Render and workspace optimization

> 🔗 **[Portal Showcase](https://medal.tv/games/screen-capture/clips/nAWRFFYqugyyWB-mL?invite=cr-MSwyZzUsNDI3MTQwMjg3)**
 * 📄 **[View Source: Math & Coordinate Matrix Library (PortalMath)](./scripts/portal-math.md)**
* 📄 **[View Source: Player Teleportation & Camera Handoff Client](./scripts/portal-traversal.md)**
* 📄 **[View Source: Procedural Ring Particle & Lighting Effects](./scripts/portal-vfx.md)**
* 📄 **[View Source: Viewport World Streaming & Terrain Extraction Client](./scripts/portal-render.md)**
---

### Security Drone System

A physics-based controllable drone built around custom flight behaviour, server authoritative gameplay and a complete deployment lifecycle.

**Highlights:**
- Custom force-based flight controller
- Acceleration, drag, banking and ground tracking
- Takeoff, landing and return-to-home behaviour
- Client/server command validation
- Local prediction for responsive interactions
- Multiple camera perspectives
- Deployment, packing and redeployment
- Failsafe and operational state handling


> 🔗 **[Security Drone Showcase](https://medal.tv/games/screen-capture/clips/nAWJQR6hVfKJZhuxB?invite=cr-MSw1eVEsNDI3MTQwMjg3)**
* 📄 **[View Source: Drone Flight Server (DroneFlightServer)](./scripts/drone-server.md)**
* 📄 **[View Source: Drone Flight Controller Physics Module (FlightController)](./scripts/drone-physics.md)**

> **Source Note:** The code provided in this portfolio consists of selected portions of the full project codebases. Some of the closely related Drone components have been consolidated into a single source page for an easier review. Additional supporting systems and project code have been kept private.

---

## Portal Development & Problems Solved
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

## Drone Development & Problems Solved

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
* **Missing Flight Remotes:** During one of my tests, the deployment script accidentally replaced the actual flight server, meaning that the remotes the drone was using such as FlightInput and DroneLook were no longer being created. Once I discovered this bug I separated the deployment and flight scripts again so they each handle their own part of the system.

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

> **Note:** I am no longer associated with either project and no longer have access to their Studio files, source code or development materials. My work on Survive Anton Chigurh's Tractor was covered by an NDA with Null_X Studio, and I no longer have files from the Blood Debt gun system. Because of this, I have listed these projects as previous experience rather than full portfolio showcases. The Portal and Drone projects above are work that I can directly demonstrate and provide source code for.


---

## Technical Notes

Below are a few short explanations of some of the more technical systems used in the projects above.

### Portal Coordinate Transformation

The portal system works by first converting a player's position or `CFrame` into the source portal's local coordinate space. The relative position and orientation can then be transformed through the portal pair and reconstructed using the destination portal's coordinate space.

This allows the system to preserve where the player is relative to the portal rather than simply teleporting them to a fixed point. It also means portals can be placed at different rotations and orientations while still producing consistent traversal.

### Momentum Preservation

When something passes through a portal, keeping its original world-space velocity would produce incorrect movement when the destination portal is rotated differently.

Instead, the velocity is transformed through the same portal-space rotation used for traversal. This means movement relative to the entrance portal is converted into the corresponding direction at the destination.

For example, downward momentum entering a portal on the floor can become forward momentum if the destination portal is mounted vertically.

### Portal Crossing Detection

The traversal system does not simply check whether a player is touching a portal.

It tracks the player's position relative to the portal plane and detects when they move from one side of that plane to the other while inside the portal opening.

Earlier versions switched state directly around `Z = 0`, which caused rapid flickering near the threshold. I separated the crossing thresholds slightly so small frame-to-frame position changes do not repeatedly trigger the traversal logic.

### Client Prediction and Server Validation

Some drone interactions are updated locally before the server response arrives so controls feel immediate.

For example, the floodlight can visually change state on the client as soon as the player presses the input, while the command is still sent to the server for validation.

The server remains responsible for deciding whether the action is actually valid. Important drone commands also check things such as whether the drone is currently deployed before being accepted.

This keeps the controls responsive without relying on the client as the authority for important gameplay state.

---

## Technical Notes & Design Decisions

Below are some additional explanations of the reasoning behind several of the larger technical decisions in the projects above.

### Why does the portal transform through the source portal's local space?

The source portal's inverse transform is used to convert the player's world-space position and orientation into coordinates relative to the entrance portal.

Once the position is to be relative to the source portal, the same relative transform can be reconstructed through the destination portal. This is what allows the system to work between portals with different positions and rotations rather than relying on amfixed world-space offset.

A 180° rotation is also applied between the two portal spaces so that travelling into the front of one portal correctly produces an exit from the front of the other.

### Why transform velocity instead of keeping the player's original velocity?

Velocity is directional, so keeping the original world-space velocity would produce incorrect behaviour when the destination portal has a different orientation.

The player's linear and angular velocity are instead transformed through the portals in the same way as their orientation.

This means momentum is preserved relative to the portals. For example, downward movement through a floor portal can become forward movement when exiting from a vertical portal.

### Why not simply teleport when the player touches the portal?

Touch detection alone does not tell the system whether the player actually crossed through the portal.

The traversal system instead tracks the player's position relative to the portal and detects when they move from one side of the plane to the other while inside the portal opening.

Earlier versions used a single threshold around `Z = 0`, but small position changes between frames could repeatedly switch the detected side. I separated the thresholds slightly to prevent this flickering. 

### Why use local prediction if the server is authoritative?

Waiting for every server response before visually updating an interaction can make controls feel delayed.

For interactions such as the drone floodlight, the client can update the visual state immediately while the command is sent to the server.

The server still validates the command and remains responsible for the actual gameplay state. This gives the player immediate feedback without treating the client as trusted authority.

### What prevents a client from controlling a packed or inactive drone?

Disabling the controls on the client is not enough because a modified client could still manually fire the RemoteEvents.

The server therefore performs its own deployment and ownership checks before accepting drone commands.

The same general rule is used throughout the system: client checks improve usability, while server checks determine whether an action is actually allowed.

### Why is the FlightController so large?

The published FlightController contains several closely related components that were consolidated into one source page for a portfolio review.

The flight simulation itself also contains tightly coupled behaviour such as movement, orientation, takeoff, landing, ground tracking, return-to-home and state transitions.

For a larger multi programmer production, responsibilities that have been compiled and can be separated cleanly could be moved into their dedicated modules, while the tightly coupled simulation logic would remain together where splitting it would make the system harder to reason about.

The complete project contains additional supporting code that is not included in the portfolio.

### How would I approach working inside an existing codebase?

When working with an existing system, I try to understand the current behaviour and architecture before changing it.

I normally trace where the relevant state originates, how it moves between scripts, and what other systems depend on it before making the smallest change necessary to solve the problem.

Afterwards I test the behaviour around the change as well as the original bug so that fixing one issue does not unintentionally create another.

A large part of my previous Roblox development work involved debugging, maintaining and modifying existing gameplay systems rather than only building new systems from scratch.
