---
layout: post
title:  "OCaml Game Of Life"
date:   2025-09-28
categories: projects
---
This semester I'm taking CS 421, which is "Programming Languages and Compilers," and the focus of the course is on the functional programming language Ocaml. So far I've found the functional programming paradigm to be a nice change of pace; there's a lot more emphasis on using recursion over while loops for iterative tasks, and a particular kind of recursion called "tail recursion" is preferred since it only requires constant stack space (whereas "non-tail recursion" takes up linear stack space).

Anyways, I wanted to play around with Ocaml on my own and get comfortable with its various features, and, more generally, with functional programming. And in order to do that, I'll be implementing Conway's Game Of Life in Ocaml. Any readers unfamiliar with the Game Of Life should take a minute to skim the [wikipedia page](https://en.wikipedia.org/wiki/Conway%27s_Game_of_Life). Essentially, there's a 2D grid, and we refer to each block in the grid as a cell. Cells can be either alive or dead, and every timestep, a cell will change (or stay the same) based on the aliveness or deadness of its eight surrounding neighbors. The only user input to this game is the initial grid state. Afterwards, the cells act autonomously.

<!-- Before we dive in, a side tangent on coding with LLMs: While, yes, they can write working code sometimes, and yes, they can save you time with the small details, you ultimately still need to have the intuition from learning the basics in order to get anywhere. From a learning perspective, asking an LLM to write your code is like going for a workout and asking the buff dude to do your set for you—the heavy lifting isn't being done by you. Instead, to continue this analogy, you'd be much better off asking the buff dude to correct your form or recommend exercise variants that might suit you better—you leverage his experience and use it to guide your journey. Now, back to our scheduled programming! (pun sooo intended) -->

I've never implemented Game Of Life before, so I started by coding in Python, a language I am quite comfortable with. Almost immediately, I ran into a bug—let's see if you can spot it.

This is a snippet of my Python code, which shows a function that computes the update that should occur given the current state of the 2D grid.

{% highlight python %}
class GameOfLife:
    ...

    def update(self):
        # each cell's action is independent of all other cells at every timestep
        # so we need each cell to have a reference to the previous state
        reference_grid = self.grid.copy()
        for x in range(self.height):
            for y in range(self.width):
                # store whether or not the cell is alive
                is_alive = reference_grid[x][y]
                
                # count how many neighbors are alive
                alive_neighbors = self.get_num_alive_neighbors((x, y), reference_grid)
                
                # a living cell with less than two living neighbors or more than 3 living neighbors dies
                if is_alive and (alive_neighbors < 2 or alive_neighbors > 3):
                    self.grid[x][y] = False
                    continue
                
                # a living cell with 2 or 3 living neighbors stays alive
                if is_alive and (2 <= alive_neighbors <= 3):
                    continue
                
                # a dead cell with exactly 3 living neighbors comes to life
                if not is_alive and alive_neighbors == 3:
                    self.grid[x][y] = True
                    continue
                
                # otherwise, do nothing
{% endhighlight %}

Did you spot it? Look right below the function declaration, the line with `reference_grid = self.grid.copy()`. Basically, I was storing the grid as a 2D array, AKA a 1D array containing other 1D arrays. By using Python's built-in `list.copy()` method, I assumed that I'd be doing a deep copy of the entire 2D array, so that no change I made to the original list would show up in the copied list. However, `list.copy()` performs a shallow copy, which creates a copy of the list you provide, but not the items inside that list. Since the items I was storing were _lists_ themselves, which are mutable in Python, I was unwittingly referencing the same contents from the original list in my "copied" list.

There's nothing like a good bug to drill an important concept into your head. Had an LLM just generated the correct code, I wouldn't have covered the gap in my knowledge, which is why I'm writing all the code by hand.

A couple hours later, the OCaml implementation is working! Honestly the time spent on this project was probably split 40/60 setting up the local OCaml environment (I tried and failed to get it to work on Windows, but luckily Ubuntu worked) and actually writing out the code. Anyways, it was definitely worth it. I practiced using `Array` functions like `fold_left` and had fun translating my programming intuition from Python over to OCaml.

Here are some highlights—the [full code](https://github.com/peterqlin/games-of-life) is on my Github.

When creating the 2D array to store the grid state, I quickly learned the difference between `Array.make` and `Array.init`—the former creates an array with copies of whatever initial item you pass in, while the latter calls a function for every element created. For `Array`s, this is especially important, as they are mutable.

{% highlight ocaml %}
let init_grid h w = Array.make h (Array.make w 0)
{% endhighlight %}

and

{% highlight ocaml %}
let init_grid h w = Array.init h (fun _ -> Array.make w 0)
{% endhighlight %}

have very different behavior, and this reflects the lesson I learned while writing my Python implementation.

At the beginning, I was hesitant to use `Array` library functions, instead opting to write out all the details myself. In this snippet, I wrote a function to get the number of alive neighbors of a certain cell.

{% highlight ocaml %}
let get_alive_neighbors cell =
  let rec num_alive neighbors =
    match neighbors with
      | [] -> 0
      | cell::cells -> let x, y = cell in if grid.(x).(y) = 1 then 1 + num_alive cells else num_alive cells
  in num_alive (get_neighboring_cells_wraparound cell)
{% endhighlight %}

I noticed that the same functionality (value-based aggregation) could be achieved with `fold_left`, so I revised my initial code.

{% highlight ocaml %}
let get_alive_neighbors cell = Array.fold_left (fun r -> fun (x, y) -> if grid.(x).(y) = 1 then r + 1 else r) 0 (get_neighboring_cells_wraparound cell)
{% endhighlight %}

By using `fold_left`, much of the boilerplate code was removed, allowing me to focus on the core functionality.

One last thing I noticed was that the constraints of functional programming forced me to think differently about how I went about my implementation. In my Python code, I took the easy route and stored an entire copy of the grid every time I needed to make an update. In OCaml, however, it wasn't obvious to me how I'd be able to replicate what I'd done in Python, and that's what helped me arrive at a better solution: storing only the cells that need to be toggled instead of the entire grid. This circumvents the issue I had in my Python code, which incrementally updated the grid, necessitating a previous grid to be stored. Now, the cells that need to be toggled are stored without modifying the grid, then the grid is updated all at once, saving a ton of space.

Sometimes the best thing we can do to make progress is to go back to basics.