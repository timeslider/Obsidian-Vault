[GDScript style guide](https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_styleguide.html)
``` GDScript
1. @tool, @icon, @static_unload
2. class_name
3. extends
4. ## doc comment

5. signals
6. enums
7. constants
8. static variables
9. @export variables
10. remaining regular variables
11. @onready variables

12. _static_init()
13. remaining static methods
14. overridden built-in virtual methods:
	1. _init()
	2. _enter_tree()
	3. _ready()
	4. _process()
	5. _physics_process()
	6. remaining virtual methods
15. overridden custom methods
16. remaining methods
17. subclasses
```