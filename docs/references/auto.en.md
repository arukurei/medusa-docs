# Automation

Programmatically creating a graph is divided into four stages: **Data Creation**, **Core Registration**, **Topology Building**, and **Logical Synchronization**.

## Basics
An atom cannot exist without an `AtomData` resource. This is its "passport" for the ProgressDB and GraphLogic systems.
When generating via code, you must either load an existing resource or create a new instance:

```gdscript
var new_data = AtomData.new()
new_data.id = &"skill_fireball" # Unique ID is required
new_data.title = "Fireball"
```

#### 1. Atom Registration (`add_atom`)

!!! warning
    Do not use the standard `add_child()`. Instead, use the `graph.add_atom(node, branch)` method.

*   The node is added to the Godot scene tree.
*   An **AtomContext** is created—an object in the graph's physics engine.
*   The atom is assigned mass, temperature, and an index for fast calculations.
*   If the atom has `AtomData`, it is automatically registered in the global `MedusaDB`.

```gdscript
var atom = atom_template.instantiate()
atom.data = new_data
graph.add_atom(atom) # Register atom in the system
```

#### 2. Building Topology (`connect_atoms`)
In Medusa, connections (lines) are more than just visual effects. They manage physics and availability logic. The `graph.connect_atoms(source, target)` method performs three functions:

1.  **Physics:** Creates an invisible "spring" (`SpringForce`) between nodes.
2.  **Logic:** Records `target` as a child of `source`, and `source` as a parent of `target`. This is critical for `ConnectionsLogic`.
3.  **Visuals:** Tells the `GraphLine` to draw a line between these points.

```gdscript
graph.connect_atoms(parent_atom, child_atom)
```

#### 3. State Synchronization (`sync_with_progress`)
After building the "web" of atoms and connections, their default status is `LOCKED`. To make the tree come alive and check which skills are now available, you must run the logic synchronization cycle.

The `graph.sync_with_progress(progress_db)` method forces the graph to iterate through all `GraphLogic` rules and update the `status` of every atom.

---
## Automation and Management

*   When creating "starter" skills via code, add the `&"root"` tag to them.
*   Configure a `TagLogic` in the Graph that allows `AVAILABLE` status for all atoms with the `&"root"` tag.
*   Now, after calling `sync_with_progress()`, your starter skills will light up automatically.

### Dynamic Manipulation (Links and Breaks)
If connections in your game can change (e.g., a dynamic relationship network), use the following methods:

*   `graph.disconnect_atoms(a, b)` — instantly removes a link and updates logic maps.
*   `graph.wake_up_atom(a)` — forces an atom out of "sleep" mode. Useful after changing links so that physics starts pushing nodes immediately.

### Focus Monitoring
For creating editors or detailed info panels, use the `graph._focused_atom` variable. 

*   This is a reference to the last atom selected by the user.
*   You can easily implement a context menu (like in the demo): `panel.position = graph._focused_atom.position + offset`.
