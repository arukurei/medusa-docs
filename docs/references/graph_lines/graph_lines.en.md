# GraphLines

In Medusa, lines are **Resources**, not nodes. This means that the same line style can be used to render thousands of connections simultaneously without bloating the Scene Tree with unnecessary objects.

1.  **Centralized Loop:** The **Graph** node, within its `_draw()` method, iterates through all registered **Atoms**.
2.  **Parameter Request:** For every "Parent -> Child" pair, the **Graph** gathers data: center positions, **Atom** radii (so the line starts at the icon's edge, not its center), and transparency levels.
3.  **Resource Call:** The **Graph** calls the `draw()` method of the assigned **GraphLine** resource, passing a `context` dictionary.

---
## Drawing Context
The `draw()` method receives a `context` dictionary, allowing the line to "understand" the state of the connection:

*   **`connection` (bool):** Is this a physical link?
*   **`navigation` (bool):** Is this a navigation link (for gamepad/keyboard)?
*   **`alpha` (float):** Current transparency (driven by **Atom** animations).
*   **`highlighted` (bool):** Should the line glow? (becomes `true` if one of the connected **Atoms** is hovered).
*   **`conn_mutual` (bool):** Is the connection bidirectional? (essential for drawing arrows on both ends).
*   **`is_primary` (bool):** A flag that prevents double rendering of the same line between two **Atoms**.

---
## Connection Modes
Almost every Medusa line style (Circuit, Soft, Dotted) features a **Connection Mode** parameter:

1.  **OFF:** The line is not drawn.
2.  **MERGED:** If there are two connections between **Atom** A and **Atom** B (back and forth), they are drawn as a single line. If the connection is mutual, arrows appear on both ends automatically.
3.  **SEPARATE:** Renders two parallel lines with a slight offset (`separate_stride`). Ideal for displaying "hostility" or "complex exchange" between nodes.

---
## Performance and Optimization

Rendering hundreds of dynamic lines can be expensive. We have implemented three levels of optimization:

1.  **Vertex Caching:** Shapes (e.g., arrows or rhombuses) are not calculated every frame. Their geometry is generated once and stored in a `PackedVector2Array`. Recalculation only occurs if the line's length or width changes.
2.  **Draw Transform:** Instead of calculating every point in world coordinates, lines use `draw_set_transform`. This allows Godot to effectively use the GPU for drawing segments.
3.  **Memory Management:** Arrays for points and colors (`PackedColorArray`) are reused between calls. This reduces the Garbage Collector (GC) load to near zero, even at 60 FPS on low-end hardware.

---
## Creating Your Own Line

You can create a completely unique connection style by inheriting from the **GraphLine** resource:

```gdscript
@tool
class_name MyPulseLine extends GraphLine

@export var color: Color = Color.CYAN

func draw(canvas: Control, start: Vector2, end: Vector2, context: Dictionary = {}) -> void:
    # 1. Calculate direction and length
    var direction = (end - start).normalized()
    var time = Time.get_ticks_msec() * 0.005
    
    # 2. Modify the point based on time (wave effect)
    var mid_point = start.lerp(end, 0.5) + direction.orthogonal() * sin(time) * 20.0
    
    # 3. Draw using standard Godot CanvasItem methods
    canvas.draw_line(start, mid_point, color, 2.0)
    canvas.draw_line(mid_point, end, color, 2.0)
```

Finally, drag your created resource into the `graph_lines` field of the **Graph** node, and all your skills will be connected by pulsating waves.
