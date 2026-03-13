# Physics

The Medusa physics engine is not a standard `RigidBody` engine. It is a **specialized graph simulation system** optimized for thousands of UI elements in real-time.

It is built on a **data-oriented approach**: all data regarding positions, velocities, and connections are packed into flat arrays. This allows for complex calculations (e.g., repelling hundreds of nodes) without FPS drops.

---
## 1. Simulation Architecture

Unlike standard physics where objects exist "on their own," in Medusa, the calculation is centralized through the **Graph** node.

Every frame, the **Graph** forms a `context` object passed to all forces. It contains:

*   `spatial_grid`: A Spatial Grid for fast nearest-neighbor lookups.
*   `centers` and `scales`: World positions and scales of all **Atoms**.
*   `radii`: Effective collision radii (accounting for **Atom** growth from connections).
*   `delta`: Frame time.

---
## 2. Calculation Phases (GraphForce)

Every force (inherited from `GraphForce`) passes through two processing stages:

#### Stage 1: Mass Calculation (`calculate_mass`)
In this stage, forces determine how "heavy" each **Atom** should feel.

!!! example
    The **InertiaForce** adds mass for each connection. This makes large "hubs" (nodes with many connections) more stable and slow, while single **Atoms** remain light and agile.

#### Stage 2: Force Application (`apply_force`)
This is where the mathematical calculation of the displacement vector occurs. Forces do not change position directly; they add values to the velocity accumulator.

```gdscript
# Inside your custom force
func apply_force(registry, forces, context):
    var dist = center_a.distance_to(center_b)
    # Calculate the vector and add it to a specific Atom index
    forces[index_a] += direction * power
```

---
## 3. Standard Force Library

To create a vibrant and organized skill tree, Medusa provides 6 core forces:

1.  **GravityForce:**
    Pulls **Atoms** toward their "gravity targets." Free **Atoms** are pulled toward the **Graph** center, while **Atoms** inside a **GraphBranch** gravitate toward the center of their branch.
2.  **RepulsionForce:**
    Works like same-pole magnets. Prevents **Atoms** from overlapping.
3.  **SpringForce:**
    Creates elastic tension between connected **Atoms**. It aims to keep them at a specific `target_length`.
4.  **InertiaForce:**
    Controls resistance to movement. Prevents the **Graph** from "shaking" during sudden changes and makes movement smoother and more physical.
5.  **ClusterRepulsionForce:**
    Treats entire **GraphBranch** units as single physical bodies. It helps huge skill clusters stay apart by pushing entire branches away from each other.
6.  **VortexForce:**
    Swirls free **Atoms** in orbital motion around the center, allowing for dynamic, rotating "star-map" skill layouts.

!!! note
    RepulsionForce uses the **inverse square law**. The closer the **Atoms**, the stronger they repel each other.

---
## 4. Optimization Systems
Physics simulation is resource-intensive. Medusa uses several technologies to ensure it doesn't drain the CPU.

#### Spatial Grid
The **Graph** divides the screen into invisible cells (size defined via `spatial_grid_size`). Instead of every **Atom** checking collisions with **all** other **Atoms** (O(n²) complexity), it only checks neighbors in its own and adjacent cells. This allows for massive **Graphs** without performance loss.

#### Thermal System (Sleep & Heat)
Medusa implements a "sleeping" node system:

1.  **Cooling:** If an **Atom's** speed falls below the `sleep_threshold`, it begins to accumulate a "freezing temperature." After a few seconds (`sleep_delay`), the **Atom** "falls asleep"—physics calculations for it stop.
2.  **Heating (Heat):** If you hover over an **Atom**, drag it, or if a moving neighbor crashes into it, the node instantly "heats up" and wakes up.
3.  **Energy Transfer:** A waking **Atom** can pass its "heat" (energy) to neighbors through connections, causing an entire branch to start moving.

!!! note
    Many engines "explode" when the scene scale (Zoom) changes. Medusa's physics automatically adapts forces to the parent's current scale. Force coefficients are recalculated so that spring tension and gravity feel the same regardless of the zoom level.

---
## 5. Creating Your Own Force

Create a new resource inherited from `GraphForce` and implement the two methods.
Example of a "Centrifugal Force" (pushes away from the center):

```gdscript
class_name OutwardForce extends GraphForce

@export var power: float = 500.0

func apply_force(registry: Array, forces: Array, context: Dictionary) -> void:
    var center = context["global_center"]
    var positions = context["centers"]
    
    for item in registry:
        var idx = item.index
        var direction = (positions[idx] - center).normalized()
        forces[idx] += direction * power
```

Then simply add this resource to the `Physics Forces` array of the **Graph** node.

!!! tip
    If atoms are too close, call `graph.wake_up_atom(atom)`. Repulsion forces will automatically position nodes at the ideal distance.
