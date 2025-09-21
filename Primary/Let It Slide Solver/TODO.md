Save the transitions and load them on start so they don't have to be recalculated each time.

I think only need to a candidate from each group. Is it possible to find a candidate from each group without exploring the whole space?

I'm making a puzzle game. The rules are as follows: The game takes place on an 8 by 8 grid. There can be empty tiles containing nothing, player tiles from 1 but more commonly many, and wall tiles while block the player's movement. When the user moves a tile, they can only move one at a time. The tile will move up, down, left, or right. It will keep going until the tile hits another player tile, a wall, or the edge of the map.

Only some starting positions are allowed. For example, on an empty board with 1 player tile, the starting positions are in the corners of the board. Starting positions can be found by using a strongly connected component algorithm. I already have such a system but it is slow. I think it's possible to do better but I'm not sure. There can be more than one group of starting positions. For example, say you have the following board:

/- - W - -
/- - - - -
/- - - - -
/- - - - -
/- - W - -

A mostly empty board but there's a wall along the top edge in the middle and one on the bottom row in the middle. For a single player tile, there are two groups. But for two player tiles, there are only one group.

I thin it would be faster, if I could just find at least one valid starting position from each group. Then I could run a BFS or DFS to find the rest. What do you think?

# Always be thinking of ways to improve the FindValidStartingPositions method

- Make sure you're only checking player tile configurations that need to be checked
	- This means we need a way of finding next lexicographically permutation with a mask.
	- PEXT and PDPT might come into play
- Use a lookup table for player movement
`0b00000000000000000011100`
