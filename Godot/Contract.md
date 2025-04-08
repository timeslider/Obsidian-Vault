A contract is an agreement between a function and node. It's like an abstract in C# but it's not strictly enforceable meaning you'll have to enforce it yourself.

---
For example, say you have a bullet, a tree, and an enemy. Everything has a collision shape. When the bullet hits a collision shape, it'll run a has_method(take_damage) on the object it hit. If that collision shape had a take_damage method, then run it, passing the amount of damage to take.

This keeps things separate. The bullet doesn't need to know about trees or enemies.

