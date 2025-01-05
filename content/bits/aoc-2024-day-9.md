---
date: '2025-01-03T21:43:56+03:00'
draft: false
title: 'AOC 2024 Day 9: A clever use of heaps'
tags: 
    - aoc
    - Advent of Code
    - Advent of Code 2024
    - algorithms
    - code
    - python
    - rust
---

I'm [participating](https://github.com/tadejsv/aoc-2024) in this year's [Advent of Code](https://adventofcode.com/2024) writing solutions in all 3 languages I am proficient in: Python, Go and C++.

I came across an interesting solution to the [Day 9](https://adventofcode.com/2024/day/9) problem. This is still one of the earlier days, so you're not supposed to pull out the big guns (fancy algos) just yet. Nevertheless, this solution (which I'll get to in just a minute) made an interesting use of min heaps to acheive a substantial speedup on part 2 of the problem.

## The problem

You have some disk space, which is populated by files, with some empty space between them:

```
00...111...2...333.44.5555.6666.777.888899
```

The numbers represent files, a group of equal numbers here is a single file, and dots are empty space. In the first part, you're supposed to move files into the empty space (starting with the rightmost file), splitting the file between multiple empty spaces if needed. Here's how the process looks like:

```
00...111...2...333.44.5555.6666.777.888899
009..111...2...333.44.5555.6666.777.88889.
0099.111...2...333.44.5555.6666.777.8888..
00998111...2...333.44.5555.6666.777.888...
009981118..2...333.44.5555.6666.777.88....
0099811188.2...333.44.5555.6666.777.8.....
009981118882...333.44.5555.6666.777.......
0099811188827..333.44.5555.6666.77........
00998111888277.333.44.5555.6666.7.........
009981118882777333.44.5555.6666...........
009981118882777333644.5555.666............
00998111888277733364465555.66.............
0099811188827773336446555566..............
```

In part 2, you can only move a file to an empty space if the whole file fits into it - no file splitting allowed. The process looks like this:

```
00...111...2...333.44.5555.6666.777.888899
0099.111...2...333.44.5555.6666.777.8888..
0099.1117772...333.44.5555.6666.....8888..
0099.111777244.333....5555.6666.....8888..
00992111777.44.333....5555.6666.....8888..
```

You'll notice that the files are numbered - these numbers are needed to compute a checksum at the end, to verify that the result is correct.

## The solution

In part 1, the solution is straightforward - you just fill empty spaces until the entire file is moved. To acheive an efficient solution, you have to track the index of the leftmost empty space available (in my solution `empty_space_ind`), so you don't have to loop from the beginning for each file.

For the first part you can represent the files and empty space either as single bytes on the line, or as blocks. I chose blocks (each block having a `start`, `length` and `index` attribute). Here's the core of the solution - the part that moves the files in the empty spaces:

```python
    empty_space_ind = 1
    for file_ind in range(len(spaces) - 1, -1, -2):
        file = spaces[file_ind]
        while file_ind > empty_space_ind and file.length > 0:
            empty = spaces[empty_space_ind]
            if empty.length > file.length: # We've moved the entire file
                file.start = empty.start
                empty.start += file.length
                empty.length -= file.length
                break

            # We completely fill this empty space
            empty.index = file.index
            file.length -= empty.length
            empty_space_ind += 2
```

For part 2, we can only move a file, if there is an empty space at least as large as itself. This means that we may skip over some empty spaces that are not large enough - however those empty spaces are still available, and may be filled later by a smaller file.

This prevents us from tracking the leftmost available empty space, so it seems that for each file, we must loop over all empty spaces from the beginning. Here's how this would look in code:

```python
    for i in range(len(files) - 1, -1, -1):
        file = files[i]
        for j in range(len(empties)):
            empty = empties[j]
            if empty.start > file.start:
                break
            if empty.length < file.length:
                continue

            file.start = empty.start
            empty.start += file.length
            empty.length -= file.length
            break
```

This solution was fast enough (takes about a second in python, and feels instantaneous in C++), so I didn't explore any further optimizations. However, when looking at others' solutions in a [reddit thread](https://www.reddit.com/r/adventofcode/comments/1ha27bo/2024_day_9_solutions/), I came upon a solution that optimized away this inefficient foor loop in part 2.

## The optimized solution

The [solution](https://github.com/maneatingape/advent-of-code-rust/blob/main/src/year2024/day09.rs) by [maneatingape](https://www.reddit.com/user/maneatingape/) (written in Rust, btw) uses an array of min heaps to store the leftmost available empty spaces.

```rust
    // Build a min-heap (leftmost free block first) where the size of each block is
    // implicit in the index of the array.
    for (index, &size) in disk.iter().enumerate() {
        if index % 2 == 1 && size > 0 {
            free[size].push(block);
        }

        block += size;
    }

    // ...
    for (index, &size) in disk.iter().enumerate().rev() {
        //...
    
        // Find the leftmost free block that can fit the file (if any).
        let mut next_block = block;
        let mut next_index = usize::MAX;

        for (i, heap) in free.iter().enumerate().skip(size) {
            let top = heap.len() - 1;
            let first = heap[top];

            if first < next_block {
                next_block = first;
                next_index = i;
            }
        }    
```

Here's how this works: the length of files/empty spaces can be no longer than 9. So we simply store available empty spaces for each of the 9 possible lengths! So when we need to place a file with size $S$ in an empty space, we check all empty spaces with size $\geq S$, and take the leftmost available space.

Min heaps are used to store the empty spaces, so that we always have the leftmost available. As there are 9 possible lengths - we just create an array of 9 min heaps.


This certainly is not a super advanced algorithm, but the usage of array of min heaps did strike me as quite clever. I don't remember seeing more than one or two heaps used in a solution.
