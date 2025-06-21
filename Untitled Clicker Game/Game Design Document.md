
---
# Plot
You're trapped on a space station stuck in a time loop. It is unknown how you got stuck. Every 15 minutes, you loop back to the beginning. However, if you can craft a piece of [[Research names#Quantum String|Quantum String]], you use it to tie down various things in your environment. For example, you can lock the recipe of a crafting machine. Anything you can set manually can be tied down. To cost of quantum strings grows exponentially, but they are a huge time saver and help you get more quantum strings.

Eventually you'll create an item to extend time

# Game loop
You order ores. Wait for them to be delivered. Once acquired, convert them into better items. You can sell them to get more ores, or spend them in research.

# How to beat the game
You're ultimate goal is to craft a fuel to speed your ship up to light speed. The faster you go, the more


# Money
The main currency in the game. Used to buy stuff.
# Power
Determines how fast machines can run. First, a `power factor` is calculated which is equal to
The total available power / total power consumed

$$\frac{\operatorname{Total available power}}{\operatorname{Total power consumed}}$$
When $\operatorname{power factor} \geq 1$, all machines run at a normal speed. When $\operatorname{power factor} < 1$, they'll run at: $$\operatorname{desired speed} \times \operatorname{power factor}$$
`power_factor` should be clamped between 0 and 1.
```
clamp(power_factor, 0.0, 1.0)
```

It should only be updated when you buy or sell something and not be recalculated each frame.

# Research

# Quantum String
# Weight
# Speed
# Time
