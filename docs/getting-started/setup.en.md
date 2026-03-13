# Setup

First of all, let's get acquainted with the core nodes of the framework. Medusa is built entirely on Godot's standard `Control` (UI) system, but it brings its own data-oriented physics engine and rendering logic to make the interface dynamic and "alive."

To create a skill tree or an interactive network, you will use three main nodes.

---
## 1. Graph (Central Controller)

> ![graph.png](../assets/images/nodes/graph.png)
> 
> The **Graph** node is the heart of the system. It acts as a space for physics simulation, an input handler, and a canvas for drawing connection lines.

!!! tip
    Any skill tree or network must have a `Graph` node as its parent or main container. It coordinates the behavior of all child elements.

- **Physics Engine:** Runs a real-time simulation (gravity, repulsion, springs). All math is optimized for UI elements.
- **Boundary & Grid:** In the `Boundary & Grid` group, you can restrict element movement using `boundary_shape` (Rectangle, Ellipse, or Custom). The `Use Grid Snap` parameter allows atoms to "stick" to a grid when dragged.
- **Selection & Focus:** Manages which element is currently selected or under the cursor. Medusa supports both mouse and gamepad navigation via the `FocusSystem` parameter.
- **Line Rendering:** The Graph automatically traverses all connections and draws lines using `GraphLine` resources.

!!! warning
    Please note that there is no border by default. If you define one using a specific style, be sure to set the `anchor_preset` and account for the graph’s dimensions: all of its elements must fit perfectly within the boundaries of this border.

### Basic Configuration:

1. **Line Style:** Assign a resource to the `graph_lines` field. You can use built-in styles or create your own by inheriting from [GraphLine](../references/graph_lines/graph_lines.en.md).
2. **Focus System:** Choose `Medusa` (the plugin's system working with pre-defined navigation keys) or `Godot` (the engine's standard automatic system based on spatial analysis). You can also set an `initial_focus_atom` which will automatically gain focus on startup.
3. **Multi-Selection:** In `Selection Mode` (Solo/Multiple), configure click behavior. For multiple selection, specify a modifier action: for example, `multi_select_action` (you can create an action in Project Settings under the Input Map).
4. **Cursor:** If you need an in-graph pointer, fill the `cursor_texture` field. On startup, the Graph will create an internal texture node. You can access it via code to gain full control over this component.

### Physics:
Medusa physics works based on a list of `Physics Forces`. Medusa provides basic physics right out of the box.

For stable behavior, it is recommended to add them in the following order:

1. **Inertia:** Gives nodes mass (heavy nodes are harder to move).
2. **Repulsion:** Pushes all atoms apart to prevent them from overlapping.
3. **ClusterRepulsion:** A special force for complex structures that pushes entire graph branches apart.
4. **Spring:** Creates "springs" between connected atoms, keeping them at a specific distance.
5. **Gravity:** Pulls atoms toward the graph center or their parent branches.

You can also add any other physics if your task requires it (for this, see [Physics](../references/physics/physics.en.md)).
Further configuration is trivial and comes down to fine-tuning each parameter to achieve more specific behavior.

---
## 2. Atom (Network Element)

> ![atom.png](../assets/images/nodes/atom.png)
> 
> The **Atom** node represents a single functional unit: a skill, item, technology, or quest.

Atoms are dynamic `Control` nodes with a circular physical representation. Their physical size is determined by the `radius` parameter, and the visual size by an icon or custom `_draw` logic.

### Connecting Atoms
Linking happens in the Inspector:
- **Connected Paths:** A list of paths to other atoms. This creates a physical connection (spring force) and a visual line.
- **Navigation Paths:** A dictionary defining neighbors for button navigation (`ui_up`, `ui_right`, etc.).

### Status System
The `status` parameter defines the logical state of the atom:

- `LOCKED`: Restricted.
- `AVAILABLE`: Ready to be purchased/unlocked.
- `UNLOCKED`: Purchased/Active.
- `HIDDEN`: Invisible (hidden by fog of war).
- `DISABLED`: Permanently deactivated.

Each status switches the active animation library specified in the atom's parameters or its data (`AtomData`).

### Animation and States
The visual feedback of an atom (hover, press, selection) is managed via the **State Registry** (`_state_registry`).

!!! tip "Technical Detail"
    To set up animations:
    - Add an `AnimationPlayer` as a temporary child of the atom.
    - Create animations with the names: `NORMAL`, `HOVER`, `PRESSED`, `FOCUSED`.
    - Save the animation library (`.tres` file) and assign it to the corresponding status slot in the atom's inspector. (To do this, save the library to the file system from the `libraries` parameter of the animation player).
    - Upon running, the atom will create an internal player and play the necessary animations when states change.

---
## 3. GraphBranch (Grouping and Hubs)

> ![graph_branch.png](../assets/images/nodes/branch.png)
> 
> The **GraphBranch** node is a container acting as a local gravity hub for a group of atoms.

Use branches to organize your tree structure:

- **Local Gravity:** Atoms inside a branch are pulled toward its center rather than the center of the entire graph.
- **Collapsing:** The `collapse()` and `expand()` methods allow you to hide or reveal entire skill branches with a smooth animation.
- **Group Dragging:** When `drag_branch_input` is enabled, you can move the entire group of atoms at once by dragging the branch background.
