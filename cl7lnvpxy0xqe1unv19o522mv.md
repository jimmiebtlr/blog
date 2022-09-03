## ML Log - 03-14-2022 - Basic graph generation environment

I'm interested in generative design, especially geometric deep learning.  

For a simple environment I'd like to have an environment where given some sizes of boxes, the network will pack the boxes as compactly as possible by outputting a set of positions for those boxes.  This output should also prefer to be square overall.

For scoring there will be 2 components to the score: 
- a penalty for boxes that are taking up the same space
- a smaller penalty for the amount of space the outputs as a whole take up

Several qualities we want in our score function
- The different loss components should scale similarly for different numbers of boxes.
- The loss for conflict should be roughly the right amount to exactly move outside the box if lr=1.
  - The reasoning for this is sgd directly on our loss function would hopefully have a less 
- Due to the previous 2 statements, our loss functions should both scale roughly with average height/width of all boxes.

On another note, our score function should be differentiable.  I'm hoping this will help reduce the amount of compute needed to train a network on the environment, but that'll be for future experiments.

## Conflict

Conflict penalty is the length of the smallest dimension's overlap.

Pseudocode
```
for each rect i
  for each rect j
    dx = abs(i.pos_x - j.pos_x)
    x_conf = maximum(0, (i.w + j.w)/2 - dx) # Only conflicts if inner term was > 0

    dy = abs(i.pos_y - j.pos_y)
    y_conf = maximum(0, (i.y + j.y)/2 - dy) # Only conflicts if inner term was > 0
```

## Footprint area

For footprint area, this is calculated by taxing the max and min of each dimension (x/y).  This is basically calculating the smallest rectangle that fits around all our boxes.

Pseudo code for calculating foot print area.
```
max_x, min_x, max_y, min_y = get_min_max(box_sizes, box_positions)

box_areas = (max_x - min_x) * (max_y - min_y) 
box_area_sum = reduce_sum(box_areas)

batch_max_x, batch_min_x = reduce_max(max_x), reduce_min(min_x)
batch_max_y, batch_min_y = reduce_max(max_y), reduce_min(min_y)
total_footprint = (batch_max_x - batch_min_x) * (batch_max_y - batch_min_y)

# A perfect score would result in an exactly square footprint, with total footprint area
# equal to the sum of areas of individual boxes.  We multiply by rectangle count and 
# average height/width to keep this scoring roughly the same as the other score functions.
footprint_penalty = (1 - sqrt(box_area_sum) / sqrt(total_footprint)) * rect_count * sqrt(box_area_sum)
```

## Important notes

The way it's implemented, our minimum will actually be with a small amount of box conflict.  I think this is fine for the moment, but can be fixed.