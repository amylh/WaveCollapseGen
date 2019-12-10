# Parallel Wave Function Collapse

### Jan Orlowski and Amy Lee
Website: [Parallel Wave Function Collapse](https://amylh.github.io/WaveCollapseGen/)
Other Material: Checkpoint | Proposal

## Summary

We accelerated a procedural image generation algorithm called Wave Function Collapse by introducing parallelism. To do so, we experimented with different CPU parallelization methods, such as using OpenMP or pthreads and different algorithms, comparing the results of each method to determine which was best. We found that we could get 2-4x speedup using openMP and also found a less-general sequential algorithm that outperformed all parallel implementations and did not parallelize well.

## Background

Wave Function Collapse is an algorithm for generating large bitmap images that are locally similar to a small reference bitmap image. The bitmaps are _NxN_ locally similar if each _NxN_ pattern of pixels occurring in the output occurs at least once in the input, possibly rotated or flipped. Patterns may be tiled (see Fig. 1) or overlapping (see Fig. 2), and may have constraints specifying which types of tiles may be adjacent to other types of tiles.

<p align="center">
  <img src="https://raw.githubusercontent.com/mxgmn/Blog/master/resources/wfc-summer-1.png"/>
  <i>Fig 1: Mapping between input patterns and tiled output pattern.</i>
</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/mxgmn/Blog/master/resources/wfc-patterns.png"/>
  <i>Fig 2: Mapping between an input pattern and local, overlapping occurrences in an output pattern.</i>
</p>

Below is a high-level description of the (sequential) algorithm, adapted from the writeup at https://github.com/mxgmn/WaveFunctionCollapse:

---
1. Read the input bitmap. Identify and count NxN patterns.
   1. For simplicity, we may format the input as a discrete set of NxN pattern patches.
2. Create an array called `wave` with the dimensions of the output. Each element of `wave` represents the state of an NxN region, or “cell”, in the output. The state of a cell encodes boolean coefficients that store information about which NxN patterns are forbidden (`false`) or not yet forbidden (`true`) for that cell. For each cell, we want to “collapse” the list of possible patterns until only 1 possibility is left. In the final output, the cell will be assigned that 1 pattern.
3. Initialize `wave` in the pristine state, i.e. with all the boolean coefficients being `true` for every cell.
4. Repeat until all cells have 1 possibility:
   1. Pick arbitrary cell(s) and randomly set one of its patterns to `true`.
   2. Reduce the possibilities of its neighboring cell depending on their _compatibility_ with its pattern. These compatibilities are defined in the input.
5. By now all the cell must be either in a completely observed state (all the coefficients except one being zero) or in the contradictory state (all the coefficients being zero). In the first case return the output. In the second case we exit without returning anything.
---

We profiled the original sequential code to identify which step was the most computationally intensive. The figure below shows the results of our investigation:

_ Insert image here _

Seeing the results, we decided to focus our efforts on parallelizing just the propagation step. 

In the C++ implementation, the model keeps track of the currently valid possibilities for tiles as a 3D array indexed by (x,y,t) where x and y are tile coordinates and t is the id of the possibility. 
Here is how the propagation algorithm is implemented in C++:

---
- Mark the observed tile as changed
- While there is any tile that is still marked as changed:
  - For each tile in the grid:
    - If the tile is marked as changed, check if any of its neighbors have their possibilities limited by the change in the tile. For each neighbor that does, eliminate those possibilities and mark the neighbor as changed. Unmark the current tile.
---

The reason parallelizing this algorithm is tricky is for the following reasons:
- Reducing possibilities in a cell affects other cells, which creates a dependency. Luckily, a change that collapses possibilities can only collapse more possibilities (can’t increase the number of possibilities). 
- The current propagation algorithm requires going over all the cells multiple times. There is no fast way of telling the number of times we need to iterate over all cells to have no remaining changes.

## Approach


## Results


## References

