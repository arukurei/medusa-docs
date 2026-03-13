# Data

Data management in Medusa is built on a **three-layer architecture**. Understanding how external databases (`MedusaDB`, `ProgressDB`) interact with the Graph’s internal registries is critical for creating high-performance and stable trees.
Medusa automatically links visual nodes to their logical data via an ID system.

## External Resources (Containers)

1.  **MedusaDB (The Catalog):**
    *   **Purpose:** Stores all `AtomData` resources currently present in the scene.
    *   **Mechanism:** Uses **Reference Counting**. When an **Atom** is added to the **Graph**, it registers its `AtomData`. If all **Atoms** with the ID `&"fireball"` are removed, the resource is automatically evicted from `MedusaDB` to save memory.
    *   **Link:** `AtomData.id` acts as the primary lookup key.
2.  **ProgressDB (Player State):**
    *   **Purpose:** Stores only the facts: what is unlocked and at what level.
    *   **Mechanism:** A simple dictionary `{StringName ID : int Level}`. This minimal data is easy to serialize into JSON or binary files for save systems.

!!! tip
    Since `ProgressDB` is a standard `Resource`, you can use Godot's built-in methods.
    **Example Save:**
    ```gdscript
    func save_game():
        ResourceSaver.save(progress_db, "user://save_progress.tres")
    ```
    **Example Load:**
    ```gdscript
    func load_game():
        if FileAccess.file_exists("user://save_progress.tres"):
            progress_db = load("user://save_progress.tres")
            graph.sync_with_progress(progress_db) # Update tree immediately
    ```
    Note: Use `.tres` extension for debugging (text format) and `.res` for the final game (binary format).*

---

## Internal Graph Registries (The Core)

When you start the game or add a node via `add_atom()`, the **Graph** doesn't just render it; it indexes the node in several internal dictionaries. This allows the **Graph** to perform lookups (e.g., "find all neighbors with the 'Fire' tag") instantly (O(1)) instead of iterating through every node (O(n)).

#### 1. ID Registry (`_id_registry`)

Maps the unique ID from `AtomData` to specific `Atom` node instances in the scene.
*   **Note:** One ID can belong to multiple nodes (e.g., if two identical items are on screen). The **Graph** will return an array of all matching nodes.

#### 2. Tag Registry (`_tag_registry`)

Caches **Atoms** by their classification tags.
*   **How it works:** If a logic rule asks to "check all nodes with the `&"root"` tag," the **Graph** doesn't search the scene; it simply pulls the pre-built list from this dictionary.

#### 3. Topology Maps (`_parents_map` and `_children_map`)

The most important part for skill trees. When you define `Connected Paths`:

*   The **Graph** analyzes the connection direction.
*   It builds a map of "Ancestors" and "Descendants."
*   The `graph.has_unlocked_neighbor()` method uses these maps to instantly determine if the current **Atom** can be unlocked based on its parents' state.

---

## Synchronization Process (Lifecycle)

Synchronization is the moment when a "static" skill tree comes to life based on player data.

**Calling `graph.sync_with_progress(progress_db)` triggers the following cycle:**

1.  **Context Update:** The **Graph** fetches the latest state from the provided `ProgressDB`.
2.  **Logic Execution:** The **Graph** runs the assigned `GraphLogic` rules for every **Atom**.
3.  **State Application:**
    *   The logic asks: "Does this Atom's ID exist in the ProgressDB?"
    *   If yes: `status = UNLOCKED`.
    *   If no, the logic checks neighbors via `_parents_map`.
    *   If an unlocked parent exists: `status = AVAILABLE`.
4.  **Visual Update:** Once the `status` changes, the **Atom** automatically switches its **AnimationLibrary**, updating its icon, color, or triggering effects (like the glow of an available skill).

!!! note
    If you create a tree programmatically, you don't need to manually change the `status` of every **Atom**. Your only tasks are to **configure the connections** and **add IDs to the ProgressDB**. The `sync_with_progress()` method handles everything else based on the `graph_logics` rules.
    **Example of automation:**
    ```gdscript
    func _on_buy_button_pressed(atom: Atom):
        progress_db.unlock(atom.data.id) # 1. Simply add the ID to the save state
        graph.sync_with_progress(progress_db) # 2. The Graph automatically determines which NEW atoms are now AVAILABLE
    ```
