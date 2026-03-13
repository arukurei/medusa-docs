# Creating Your Game

Medusa separates content into three levels: **Data** (Resources), **Logic** (Rules), and **Representation** (Nodes in the scene). To create a working skill tree or interactive graph, you need to connect these elements together.

!!! tip
    Always call `await get_tree().process_frame` in `_ready()` before working with the graph. The system needs time to index all connections.

---
## 1) Databases and Progression
The system relies on two types of databases. This allows you to easily save player progress without affecting the configuration of the skills themselves.

##### MedusaDB (Global Catalog)
This is the "warehouse" for all elements available in the game.

- Each object corresponds to an **AtomData** resource.
- **MedusaDB** stores links to these resources and maps them to unique `id`s (StringName).
- When you add an `Atom` to the graph and assign it an `AtomData`, the graph registers this resource in the database.

##### ProgressDB (Save File)
This is a dynamic dictionary that tracks player progress: `id : level`.

- If a skill ID is in this database, it means it is unlocked.
- The `progress_db.unlock(id)` method is the primary way to "purchase" a skill in code.

For more details, see the [Data](../references/data/data.en.md) section.

---
## 2) Graph Logic (GraphLogics)
**GraphLogic** is a powerful rule system that automatically manages atom statuses (`LOCKED`, `AVAILABLE`, `UNLOCKED`, etc.). Instead of manually changing the color of every button, you define a set of rules for the entire Graph.

The Graph has a `graph_logics` array. When `graph.sync_with_progress()` is called, the graph iterates through every atom and asks each rule: "Can this atom be unlocked?".
Rules return one of three types of answers:

1.  **BLOCK:** If at least one rule returns `BLOCK`, the atom becomes `DISABLED` or remains `LOCKED`.
2.  **ALLOW:** If a condition is met (e.g., a parent skill is purchased).
3.  **PASS (Neutral):** The rule has no opinion (e.g., the skill doesn't have the tags the rule is configured for).

#### Built-in Logic Types:

- **ConnectionsLogic:** Checks for open connections. In `ANY_PARENT` mode, a skill becomes available if at least one "ancestor" is unlocked.
- **ProgressLogic:** Checks if the skill is already purchased to set its status to `UNLOCKED`.
- **TagLogic:** Allows "root" skills (`Root`) to be available by default by assigning them a specific tag.
- **BlockLogic:** Hard-blocks branches if specific conditions are met.

To create custom logic, see [GraphLogics](../references/graph_logics/graph_logics.en.md)

---
## 3) Development: Dynamic and Static Graphs
Medusa supports two approaches to creating trees:

##### A. Manual Creation (In Editor)
You place `Atom` and `GraphBranch` nodes in the Godot interface, configure their `Connected Paths`, and set up visual effects. This is ideal for small, fixed skill trees.

##### B. Programmatic Generation (Graph API)

1.  **Atom Spawning:**
    Use `graph.add_atom(atom_node, branch)`. This registers the atom in the physics system.
2.  **Linking:**
    The `graph.connect_atoms(parent, child)` method creates a physical spring and logical hierarchy.
3.  **Synchronization:**
    Call `graph.sync_with_progress(progress_db)` after any change to recalculate all node states instantly.
4.  **Focus Management:**
    Use `graph._focused_atom` to track user interaction for floating UI elements (like `ctx_panel` in the demo):
    ```gdscript
    ctx_panel.global_position = graph._focused_atom.global_position + offset
    ```
5.  **Mass Operations:**
    Using `graph._selected_atoms`, you can link or delete entire groups of elements in one click.

For more details, see [Automation](../references/auto.en.md)
