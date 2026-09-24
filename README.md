# Whitewolf7658
**Contact:** [Your Discord Handle Here] | [Link Your Roblox Profile Here]  
*My Main Focus Areas: 3D Coordinate Transformations, Physics Solvers, and Asynchronous Memory Optimizations, Additionally I love bug hunting and improving scripts as those are my strong suits.*

---

## 🛠️ About Me
When I first set out to make my very first game back in early 2023, I was quickly rudely awoken to the complex or what I thought at the time complex systems of Roblox Studio and scripting in general! I was completely clueless at the time and confused about just how to use Studio's tools, how to make systems, and how to make them communicate properly. 

Instead of quitting, I decided to focus entirely on learning how programming actually works. I spent countless hours trying to drill the basic concepts in, slowly I saw progress, and as I started moving onto the more advanced side of development. During this, I had told one of my close friends about what I had been doing and my interest in making my own game one day, eventually my friend introduced me to a group of his friends who are currently in university for computer science and are highly experienced in game scripting as they do it themselves, some of them have even made both Indie games on steam and games on Roblox! They essentially became my tutors while I tried to learn, they helped by guiding me through the complex mathematical and the backend design concepts of scripting. They were a crucial part in me being where I am today.

For almost the last three years, I have dedicated 2 to 3 hours daily (Or tried to some days) to truly understanding scripting with the help of what I see as my mentors. My routine had relied heavily on a constant loop. This may have been learning a concept that was hard for me at the time (Things like mapping rotated 3D spaces, calculating an exact object intersections, or handling custom physics momentum) and consolidating my knowledge through writing Luau code to verify my learning, I would then figure out the errors and how to fix them, I would then try translating that knowledge directly over to Roblox with the help of the group my friend introduced me to.

The two systems I have documented below are built completely from scratch other than the Drone Model and the Ring Model as I do not specialize in modeling, they focus heavily on server security, my knowledge of network efficiency, and memory stability so they can work reliably within highly populated servers.

---

## 🌌 Project 1: The Quantum Portal System
A custom spatial transformation framework designed to handle non-Euclidean geometry, perspective translation, and manual momentum preservation between moving parts.

### Technical Overview
* **Continuous Collision Detection (CCD):** Implements a custom parametric line-segment intersection loop running in `PreSimulation` to compute exact plane crossings, replacing the default engine `.Touched` event to avoid desynchronization.
* **Unified Transformation Matrix:** Utilizes a stateless transformation matrix derived from matrix inversion (`CFrame:Inverse()`) to calculate relative object offsets and translate linear/angular velocities flawlessly across unaligned angles.
* **Asynchronous Level Streaming:** Optimizes client memory by using a time-sliced batch loop to extract raw voxel data arrays (`ReadVoxels()`) and separate static world elements from dynamic updates into a throttled 30Hz cycle.

### Showcase & Mechanics
> 🔗 **[Click Here to Watch the Portal Gameplay Showcase](PASTE_YOUR_VIDEO_LINK_HERE)**

---

### 🛠️ Development Logs & Solved Bottlenecks
This system underwent multiple revisions (from V1 to V3) to identify and eliminate edge cases, visual artifacts, and CPU performance drops. Below are the core technical problems that I had solved during the Portals development:

*   **Virtual Camera Obstruction:** Moving the virtual viewport camera backward to establish accurate depth perception caused it to render the back geometry of the wall holding the exit portal. Solved by writing a localized occlusion filter to hide the `DestinationHost` within that specific frame container.
*   **Perspective Scaling Distortion:** Initial attempts to solve character framing via wide Field of View configurations (100°+) created barrel distortion. Solved by adjusting camera placement to a fixed 8-stud distance paired with a balanced 52° lens.
*   **Frustum Clipping Glitch:** Because backing host walls were hidden inside the viewport, character avatars standing behind the virtual camera position were caught in the rendering field, obstructing the view window. Solved by writing an automated directional clip-plane culling track via local Z-depth calculation.
*   **Threshold Position Flicker:** Evaluating character rendering states at a strict `Z = 0` line created visual stutter due to micro-rounding variations between frames. Solved by implementing mathematical hysteresis, mapping separated hide/show boundaries at `-0.15` and `0.15` studs.
*   **Infinite Traversal Re-Entry:** Teleporting a player directly onto the destination plane instantly re-triggered the inverse calculation, trapping the avatar in an infinite loop. Solved by implementing a `blockedPortal` memory state and an asymmetric `REENTRY_DISTANCE` safety radius.
*   **Physical Aperture Obstruction:** Solid backing geometry frequently blocked avatars even when the visual portal window was active. Solved by tracking local player coordinates to dynamically toggle `CanCollide = false` on the `HostSurface` within a localized bounding box.
*   **RenderStepped Performance Spikes:** Forcing static world geometry to update CFrames and properties 60+ times per second was resource-heavy. Solved by isolating permanent assets and throttling dynamic update passes down to a 30Hz accumulator register.
*   **Global Array Scan Bottlenecks:** Sweeping the entire workspace array via `GetDescendants()` every 0.5 seconds for part replication heavily lowered frame rates. Solved by discarding global searches for high-speed key-value cache tables mapping original parts straight to their cloned variants.
*   **Voxel Mesh Blockiness:** Initial voxel terrain extraction resulted in a blocky, stepped grid appearance along smooth terrain formations. Solved by tightening extraction parameters down to a high-fidelity 4-stud surface shell while dropping hidden interior voxel grids.

---

