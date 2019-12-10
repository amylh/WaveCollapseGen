# Parallel Wave Function Collapse

### Jan Orlowski and Amy Lee
Website: [Parallel Wave Function Collapse](https://amylh.github.io/WaveCollapseGen/)
Other Material: Checkpoint | Proposal

## Summary

We accelerated a procedural image generation algorithm called Wave Function Collapse by introducing parallelism. To do so, we experimented with different CPU parallelization methods, such as using OpenMP or pthreads and different algorithms, comparing the results of each method to determine which was best. We found that we could get 2-4x speedup using openMP and also found a less-general sequential algorithm that outperformed all parallel implementations and did not parallelize well.

## Background

Wave Function Collapse is an algorithm for generating large bitmap images that are locally similar to a small reference bitmap image. The bitmaps are _NxN_ locally similar if each _NxN_ pattern of pixels occurring in the output occurs at least once in the input, possibly rotated or flipped. Patterns may be tiled (see Fig. 1) or overlapping (see Fig. 2), and may have constraints specifying which types of tiles may be adjacent to other types of tiles.

<p align="center">
  <p><img src="https://raw.githubusercontent.com/mxgmn/Blog/master/resources/wfc-summer-1.png"/></p>
  <i>Fig 1: Mapping between input patterns and tiled output pattern.</i>
</p>

<p align="center">
  <p><img src="https://raw.githubusercontent.com/mxgmn/Blog/master/resources/wfc-patterns.png"/></p>
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

<p align="center">
  <img src="https://raw.githubusercontent.com/amylh/WaveCollapseGen/master/texts/graph-profiling.png"/>
</p>

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

For simplicity, our project only dealt with the case where output patterns are tiled, rather than overlapping. However, the techniques we used could easily be extended to work for both.
As starter code, we used a sequential C++ implementation of the WFC algorithm by Emil Ernerfeldt. For parallelization, we used OpenMP, C++ pthreads, and the boost library. Most of the work went into modifying the propagate function to parallelize it, but we also had to modify parts of the code for running convenience.

### Multi-core Parallel with OpenMP

For this approach, we tried to parallelize the `propagate` function using OpenMP.

A key observation here is that, when tiles are propagating their states to the next iteration, they depend on the current states of their neighbors. However, each tile can check the states of its neighbors independently during each round of propagation, which is amenable to parallelization.

The propagation step can be broken down into several levels of iteration: the program iterates over every tile in the image; each tile iterates over its four neighboring tiles to check whether any of them changed state; each neighboring tile that changed state then iterates over the set of possible patterns to check whether any of them are still possible for the tile.

First, we parallelized the tile iteration. Initially, we enabled the assignment of one thread to each row of tiles in the image, and saw roughly 2-4x speedup compared to the sequential version. However, since each tile can check its neighbors independently, we decided to instead allow assignment of one thread to every tile in the image. Speedup was still roughly 2-4x, but was slightly faster than the row-wise assignment. 

We also theorized that each tile would be responsible for roughly the same amount of work, as each tile has a fixed amount of neighbors (between two to four) and patterns to check, so we used a static assignment of threads. However, we found that using a dynamic (guided) assignment led to slight speedup compared to the static assignment, implying that work variation among tiles may actually accumulate after many rounds of propagation and lead to some work imbalance. 

Finally, we considered nesting parallel loops to account for the iterations over neighbors and patterns. However, these led to poorer results than using a single parallel loop.

### Multi-core Parallel using Lock-free queues and OpenMP

For this approach, we continued to parallelize the `propagate` function using OpenMP. However, we introduced further optimization by utilizing work queues.

A key observation is that, in our case, if a tile changes only its neighbors are affected. So, instead of looping over every single tile to check if any changes need to be propagated, we instead keep a queue of tiles that could change. Whenever we process a tile and find that a change occurred, we add its neighbors to the queue. We finish once the queue is empty (i.e. we finished propagating). This gives us a sequential algorithm, but also a new avenue for parallelization. Using a lock-free queue, we can have multiple threads grab tiles from the queue, process them and add new ones onto the queue if necessary until the queue is completely empty. Our hope would be that the contention from all threads accessing the queue would not slow down the execution too much and we will benefit from more threads processing tiles.

To set up the initial parallel algorithm, we used the `boost` library’s lock-free queue as the queue and also used an `std::atomic_int` to set up a special barrier. The barrier was added to solve a problem: when the queue is empty, it does not necessarily mean a thread’s work is done. It is possible that another thread could be processing a tile and will add the tile’s neighbors to the queue once finished. So, we should only finish when all threads see that the queue is empty. The `barrier` int is incremented when a thread is working and decremented once when a thread sees the queue is empty. If a thread checks on the queue and finds it is not empty, it increments the `barrier` again and keeps working.

### Multi-core Parallel with pthreads

For this approach, we tried to parallelize the `propagate` function using pthreads. As opposed to OMP threads, where the assignment of threads to data is unknown to us, we theorized that we could explicitly assign pthreads to chunks of data in a manner that would take advantage of caching.

Our first approach was to statically interleave the pthreads and their assigned tiles. We then tuned the number of pthreads used to determine whether it influenced performance. However, this approach does not take advantage of caching, as interleaving the threads scatters the tiles that a thread works upon.

Our second approach was to statically assign the pthreads to blocks of tiles. We theorized that a blocked assignment would introduce some work imbalance, but could allow for better caching behavior compared to an interleaved assignment.

### Other Approaches

Besides the approaches listed above, we also attempted to implement parallelization using ISPC tasks and CUDA. We were unable to produce working implementations due to incompatibilities in our machine configurations and base code, but believe that further research could yield promising results.

## Results


## References

