---
title: "Puzzle cube"
date: 2023-12-17
excerpt_separator: <!--more-->
---

![Puzzle cube](/images/2023-12-17-puzzle-cube.png){:width="50%" height="50%"}

Some time ago this fascinating cube appeared in the lunch room at the office and I'll tell you, it's probably one of the more frustrating puzzles I've encountered. You feel you have everything under control the first couple of pieces, however, towards the end you realize none of them fit and you end up like in the situation above. You were so close this time so you feel you have no other choice than to go again...

<!--more-->

I never managed to solve the cube by hand and it may have been my patience that ran out, but the more likely cause is that solving puzzle cubes is not a part of my job description (sadly). But the cube was still lingering in my mind and in the middle of all christmas preparations I decided to take a crack at it, but this time with the help of a computer.

To still keep it a bit interesting I kept myself from googling too much about the problem, since I was sure someone had solved it before, and indeed they had so I'll post a reference in the end. To start of I'll give my fresh perspective to the problem.

## The puzzle

![Puzzle pieces](/images/2023-12-17-pieces.png){:width="50%" height="50%"}

The puzzle is fairly simple, given a set of pieces you are supposed to build a 4x4x4 cube. Above is a picture of all the pieces, sorry for the poor photos but they were taken rather hastily as I was leaving the office on a Friday evening. There are 13 pieces in total, all with a unique shape.

You are free to put them in any way you want, and the colors of the pieces should have no significance for the actual solutions. The puzzle is completed when you have placed all pieces in such a way that there are no holes or pieces sticking out.

Sounds simple enough!

## The problem

However, when you start looking at the problem in detail you see it for the combinatory nightmare it is. If we just consider the ways to place a single piece into a cube, there are up to 24 unique orientations of the piece and with 

There are 13 pieces, each with up to 24 possible unique orientations. Combine this with the all the possible positions to place the cube. I wrote some rough code just to generate all the permutations of each piece and in the worst case there were 432 permutations for a single piece. With 13 pieces, doing an exhaustive search on all the combinations means you have to go through somewhere around `10^{30}` permutations, maybe a bit less but way too many nonetheless.

Luckily we have some contraints that can elimate a large number of permutations. Puzzle pieces can not overlap eachother, and you are not allowed to have pieces sticking out of the cube. As mentioned early on, a big frustration of the puzzle was that half-way through you realize the current configuration of pieces simply forbids you from building further. At this point, you eliminate any permutations that branches out from this configuration.


## Setting up

The first thing I did was to actually input all the pieces into a format my future solver would be able to parse. For simplicity I decided encode each piece as a bitmask. Each bit being a element of the cube, which in total holds `4x4x4=64` bits. 1 marks bits occupied by the piece, and 0 is empty space.

I didn't do anything fancy for the actual input, I just manually "drew" the pieces from the picture above in a .txt:
```
# 0
0100
1110
0100
0000
0000
0000
0000
0000
```
Since each piece was a maximum of 2 bits in depth, I decided to restrict each input to just be `4x4x2`. Briefly explained:
```
z y x: 0123
0 0    0000
0 1    0000
0 2    0000
0 3    0000
1 0    0000
1 1    0000
1 2    0000
1 3    0000
```

I generated all possible permutations of each piece by rotating it on the X, Y, and Z axis, then translating it around the cube to cover all possible positions. Important aspect here is to remember to eliminate any duplicates produced, or permutations that are considered illegal (bits outside the actual block).

So this is what I have so far:
* 13 pieces
* For each piece, a set of possible placements in the cube (around 48 to 432 per piece).

## Search

Some general aspects of the search implementation. As the cube was 4x4x4 it was very convienient to represent the cube as a 64 bit unsigned integer. So for simplicity, this is how the cube, and also the pieces are represented in all code above. This means we can very easily implement certain operations using bit operators:
* Check that two cubes/pieces have not overlap: `current_cube & piece == 0`
* Add piece to cube: `current_cube | piece`.
* Count number of filled bits: [popcount!](https://vaibhavsagar.com/blog/2019/09/08/popcount/), or `.count_ones()` in Rust.
* and other various bit goodies: [C++](https://en.cppreference.com/w/cpp/header/bit), [Rust](https://doc.rust-lang.org/std/primitive.u64.html)


### First attempt

For my first attempt I just tried a naive recursive approach that looked something like this
```rs
fn search(state: u64, piece: usize, placements: &Vec<Vec<u64>>) {
    if piece == 13 {
        print("Found!")
        return;
    }
    for placement in placements[piece].iter() {
        if state & *placement == 0 { // No overlap
            search(state | *placement, piece + 1, placements)
        }
    }
}
```
This was not fast... The main issue being that it did not end the search early enough. Very similar to how a human would play, it usually did not realize it was game over until it had around 2-3 pieces left. I started adding some contraints to it, like limiting the number of placements to go through based on previous state but the code got a bit too complicated for my liking so I gave up.

### Second attempt

This time I decided to flip the problem around. Lets consider the task of filling the the cube bit by bit. For a complete solution we must fill in all 64 of them.

To start of, a task that is very quick in comparison is creating a lookup for all 64 bits, which maps the bit to a set containing all the pieces and placements that happen to cover that particular bit. So now we have a monstrous vector-in-vector-in-vector (`bit_map[64][NUM_PIECES][...]`).

Now, the search task is a bit more contained and the map above eliminates a lot of the possibilities we had to go through in the previous attempt:

```rs
fn search(state: u64, used_pieces: u32, bit_map: &Vec<Vec<Vec<u64>>>) {
    if used_pieces.count_ones() == 13 {
        // All pieces have been placed!
        print("Found!")
        return;
    }

    // Find first non-set bit in our cube
    let bit_index = state.trailing_ones() as usize;

    for piece in 0..NUM_PIECES {
        if used_pieces & (1 << piece) != 0 {
            // Already used
            continue;
        }

        for permutation in bit_map[bit_index][piece].iter() {
            if state & *placement == 0 { // No overlap
                search(
                    state | *placement,
                    used_pieces | 1 << piece,
                    bit_map
                );
            }
        }
    }
}
```
Using this the solution started rolling in very quickly (1000s in just a few seconds), but I've not let it run all the way to completion yet to see how many solutions it generates in total. Might also have to remove some duplicates and so on to get an accurate number of the possible solutions.

## Conclusion

My main goal was really just to generate a single solution so that I could end all my colleagues frustration once and for all, but as you go deeper there are so many other interesting aspects. As with all these types of problems, they are usually bigger then they look. Just the fact that there are thousands of solution is hard to imagine by just looking at the cube.

I've put all the code related to this on my Github: https://github.com/simeks/bedlam-cube, it might not be in great shape but feel free to check it out. It's all written in [Rust](https://www.rust-lang.org/).

Some things that I would like to look at in the future, but probably won't:

* The search algorithm, I've only briefly grasped on combinatorics so I'm far from an expert when it comes to these types of problems but it would definitely be interesting to learn more. I suspect there are a lot of shortcuts you can take.
* Performance aspects, having the `u64` representation and a lot of nice instructions for working with bits I suspect there are a lot of things you can do to speed things up. The memory structure is also very hastily put together so might be things to improve there as well.

And finally, as promised, some related work: a colleague linked me similar work by a Scott Kurowski which ~~has~~ had some great writings on problems like this [here](https://web.archive.org/web/20220328162256/http://www.scottkurowski.com/BedlamCube/). After some more digging I even found a [wikipedia article](https://en.wikipedia.org/wiki/Bedlam_cube) that could've saved me a lot of time, but that's no fun :-).

#### Update

I have some resuts! After filtering out some duplicates (the search would produce the same solution for each cube orientation), the search algorithm produced **19,186** unique solutions! And it took around 90 minutes to find all of them, which isn't too shabby. This matches the previously [mentioned article](https://web.archive.org/web/20220328162256/http://www.scottkurowski.com/BedlamCube/), which is a good sign! That article also mentioned it took 86 hours on a 2 GHz Pentium 4, so this makes you really appreciate modern hardware.

