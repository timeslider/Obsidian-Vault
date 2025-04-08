---
tags:
  - godot
  - saving
  - loading
---
---
SaverLoader shouldn't know about the internals of any node.

## Saving
---
### Collection phase
Sends an event that the game is about to be saved. It gives everything that is savable a list to write down what it wants to save
- Everything that wants to save should have a [[Contract]]
- `func on_save_game(saved_data: Array[SavedData])`
- 
### Saving phase
Save everything in the savable list to file
## Loading
### About to load
Clear the stage
### Loading phase
Loop through each element in the save file
Create a new node and then give this node its data and have the node restore itself

