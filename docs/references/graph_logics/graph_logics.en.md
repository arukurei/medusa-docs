# GraphLogic

The **GraphLogic** system is the brain of your Graph. It allows you to automate the behavior of hundreds of nodes without manually writing conditions for each one individually. Below is a detailed breakdown of how to create your own rules, how priorities work, and important nuances to consider.

Any logic script inheriting from `GraphLogic` functions as a filter in a pipeline. When the Graph updates, it runs every **Atom** through this pipeline of rules.

## 1. The `get_permission` Method: Who has the right?
This method answers only one question: "What is this rule's opinion regarding this specific Atom?". It does not change the Atom's state; instead, it returns a `Return` value:

*   **`Return.BLOCK` (Priority 1):** A hard veto. If at least one rule returns `BLOCK`, the Atom will never become available, even if other rules are "FOR" it. Use this for "mutually exclusive talents" or story-based restrictions.
*   **`Return.ALLOW` (Priority 2):** Permission. the rule says: "I believe this Atom can be opened." For an Atom to become `AVAILABLE`, it needs at least one permission in the absence of any blocks.
*   **`Return.PASS` (Priority 3):** Neutrality. The rule says: "I don't care, let the other rules decide." This is the default value.

!!!example
    If no rules on the Graph return `ALLOW`, all Atoms will remain `LOCKED` because no explicit permission was granted.

---
## 2. The `apply_status` Method: Controlling Visual State

While `get_permission` simply returns a status, the `apply_status` method is where real changes to Atom properties (`status`, `modulate`, `visible`, etc.) happen.

**Extended Logic Example:**
```gdscript
@tool
class_name HiddenFogLogic extends GraphLogic

func apply_status(atom: Atom, graph: Graph, progress: ProgressDB = null) -> bool:
	# If this is a "Secret" skill and it has no unlocked neighbors
	if atom.tags.has(&"secret") and not graph.has_unlocked_neighbor(atom, progress):
		atom.status = Atom.Status.HIDDEN
		atom.visible = false # Physics still works, but it's invisible to the player
		return false # Stop the chain so other rules don't set it to LOCKED or AVAILABLE
	
	atom.visible = true # Show if conditions are met
	return true # Allow subsequent rules (e.g., ProgressLogic) to refine the status
```

!!! warning
    **Order matters:** Rules in the `graph_logics` array are executed from top to bottom.

!!! note
    The method returns a `bool`. 
        Returning `false` tells the Graph that this rule was critical and it should **stop** querying subsequent rules for this specific Atom. 
        Returning `true` allows the Graph to continue to the next rule in the list.

---
## 3. Nuances and Tips
Use the `atom.get_all_tags()` method. It combines tags defined manually in the `Atom` node with tags stored in the `AtomData` resource. This allows you to create rules that work for specific instances as well as entire classes of resources.

#### Using ProgressDB in Logic
Your logic has direct access to the `progress` database. You can check more than just "is it unlocked"; you can check levels:

```gdscript
# Inside get_permission
if progress.get_level(&"base_intelligence") < 5:
    return Return.BLOCK # Intelligence is too low for this skill
```

#### The `graph.has_unlocked_neighbor()` Method
This is the "golden method" for skill trees. It automatically checks all parents (or children, if you specify `ConnectionDir.OUTBOUND`) and tells you if at least one of them is unlocked in the progress database.

## Rule Order in the Graph (Best Practice)
For a stable skill tree, arrange your `graph_logics` as follows:

1.  **BlockLogics** (Deny anything that is forbidden under any condition).
2.  **SpecialRules** (e.g., `TagLogic` for the roots of the tree).
3.  **ConnectionLogic** (Checking connections: can the next skill be bought?).
4.  **ProgressLogic** (Final check: has this skill already been purchased?).

This order ensures that a purchased skill always remains `UNLOCKED`, but if the path to it isn't reached yet, it remains `LOCKED`. If a story-based `BlockLogic` disables it, it correctly becomes `DISABLED`.
