---
layout: post
title: "Life in Clojurescript"
date: 2014-11-16 23:00:00
---


I've never really done more than dip my toe into Javascript development before, but the browser as a computing platform is interesting to me. Browsers have great live editors, they're visual, they're everywhere, and they're resource constrained. Javascript is an asynchronous language running in a single-threaded platform. I'd watched some talks on React and Om the idea of a virtual dom that batches updates was intriguing. So I thought I'd jump in and start with canvas.

#_"Canvas will probably be the bottleneck, life runs in linear time!"_

I decided to start with Conway's life. It's been around for awhile, it's a simple graphical exercise. I thought it would be easy to get a component to render, and it was. I also thought it would be easy to get running in the neighborhood of 30 frames per second. It wasn't.

So I set myself a challenge: write an implementation of Life in Clojurescript that runs reasonably fast and doesn't lean on mutation of state. If I wrote a program that couldn't, in concept, run concurrently, it was out.

#_So naive_

Let's take a look at the standard formulation of Life:

 - There is an n x n field of cells.
 - A cell is either _alive_ or it is _dead_.
 - A cell has a _neighbor_ if one of the eight cells adjacent to it is alive.
 - At each step of the simulation, we generate a new state based on the current one, which has the following rules:
   - If a cell is dead and has exactly three neighbors, in the succeeding step, it will be alive.
   - If a cell is alive and has less than two neighbors, it dies in the succeeding step.
   - If a cell is alive and has more than two neighbors, it dies in the succeeding step.

In most languages, this problem strongly suggests a naive solution. Each step is an n x n array of boolean values, over which we iterate and generate the next state. It didn't end well.

So let's start with initialization:

{% highlight clojure %}

(defn init [width height cells]
 (let [state (->> (repeat height (into []
                                  (repeat width false)))
              (into []))]
  (reduce (fn [acc x]
           (toggle-cell acc x)) state cells)))

{% endhighlight %}

So the first thing this does is separate the index of an element from its values, so we'll need a couple of functions to do that translation for us.

{% highlight clojure %}


(defn toggle-cell [state [x y]]
 (update-in state [x y] not))

(defn get-cell [state [x y]]
 (get-in state [x y]))

{% endhighlight %}

Already I'm not crazy about this, but let's press on. Now we'll need to get the neighbors.

{% highlight clojure %}


(defn neighbors [x y]
  (for [dx [-1 0 1]
        dy (if (zero? dx) [-1 1] [-1 0 1])]
    [(+ dx x) (+ dy y)]))

(defn neighbor-count [world [x y]]
  (->> (neighbors x y)
       (map #(get-cell world %))
       (filter #(= true %))
       count))

{% endhighlight %}

This provides everything required to build out the next generation.

{% highlight clojure %}

(defn stepper []
  (fn [state]
    (let [width (count world)
          height (-> world first count)
          locs (cells width height)]
      (reduce (fn [acc loc]
                (let [neighbors (neighbors-count world loc)]
                  (cond (and (false? (get-cell world loc))
                             (= 3 (neighbors world loc)))
                        (toggle-cell acc loc)
                        (and (true? (get-cell world loc))
                             (or (< neighbors 2)
                                 (> neighbors 3)))
                        (toggle-cell acc loc)
                        :else acc))) world locs))))

{% endhighlight %}

Great, let's fire it up on a 100 by 100 grid and see how it works! We'll run
a 60 second bench mark and see how many generations we can create.

{% include life_sim.html id="first" %}

...Oh. It's slow. Just real, super slow.

Cracking open the Chrome developer tools, we can run a little benchmark to see where
the problem areas are, and it looks like it's in neighbor-count, specifically in the
counting function. Count itself is pretty fast, clocking in at 2.1ms this run, but it's
called once per cell, which in this simulation means that it's called 10,000 times.

We're referring to each neighbor 9 times so we could do some fancy caching each generation,
but that still doesn't gain us too much.

#_Seek Simplicity_

In his *Graphics Programming Black Book*, Michael Abrash tackles life as well. Ultimately his
implementations rely on bit-banging mutable state, so the details of his optimizations aren't
that useful to us. But, after his first attempt he makes the point that sometimes, we can reap
great gains by making the problem *simpler*. He encourages us to step back and think about the
problem a little more, and bring our right brain to bear on the problem.

Here are some things we notice after messing around with Life a bit.

 * Most of the time, the majority of cells are dead.
 * State changes happen infrequently.

Let's tackle the first point. The Clojure community holds simplicity in high regard, and for a 
problem like this one, surely there's a very simple implementation of the problem.

As luck would have it, Christophe Grand wrote a very simple implementation!

{% highlight clojure %}

(defn neighbours
  "Given a cell's coordinates, returns the coordinates of its neighbours."
  [[x y]]
  (for [dx [-1 0 1] dy (if (zero? dx) [-1 1] [-1 0 1])]
    [(+ dx x) (+ dy y)]))

(defn step
  "Given a set of living cells, computes the new set of living cells."
  [cells]
  (set (for [[cell n] (frequencies (mapcat neighbours cells))
             :when (or (= n 3) (and (= n 2) (cells cell)))]
         cell)))

{% endhighlight %}

Yeah, that's it. It's tiny! More importantly, it allows us to focus on only the *live cells*
rather than the entire board state. The step function takes the neighbors of each live cell and
counts the frequency with which these neighbors appear. If a neighbor appears three times, then
it must be alive in the next generation, and if it appears twice and is already alive, it stays alive!

Great, let's see how it runs. The speed with which this one runs depends heavily upon how many cells are
alive throughout the simulation, so let's put in 5000 cells and see how we fare.

{% include life_sim.html id="second" %}

Not great, but not bad! Certainly better than the first implementation. Let's see if
we can't speed it up some.

The first thing I notice is that neighbors is being called on every cell, but its output isn't dependent
on board state. Let's memoize it and see if that helps at all.

That nearly doubled the output! The other bottleneck appears to be frequencies.
It's already implemented using transients, so while the frequency map is being
generated it's being done to mutable state. We never see it so it's okay, but
I can't think of a way to make it faster.

#_Limit Change_

So let's just use a litte bit more right brain and address Abrash's second point.
**State changes happen infrequently**.

This one took some thinking. I definitely went around in circles on it a couple of times,
especially trying to think about the advantages we might get from immutable data. Since
I've been building the benchmarks in om, I got to thinking about one of the big wins in
om and React: diffing.

So I introduce a third property I've noticed about Life: **State changes in the next 
generation depend heavily on the previous one**.

Think about it: If none of the neighbors of a given cell change this generation, that
cell will not change in the *next* generation!

So we have to sacrifice a little bit of the simplicity of Grand's solution in the service
of capturing the *changes* from the last generation.

{% highlight clojure %}


(defn neighbors [[x y]]
  (for [dx [-1 0 1] 
        dy (if (zero? dx) [-1 1] [-1 0 1])]
    [(+ dx x) (+ dy y)]))

(defn changed-cells [[x y]]
  (for [dx [-1 0 1] 
        dy  [-1 0 1]]
    [(+ dx x) (+ dy y)]))


(defn step [state]
  (let [{:keys [cells changed]} state
        changed-neighborhood (set (mapcat changed-cells-memo changed))
        changed-and-alive (s/intersection changed-neighborhood cells)
        changed-cells (frequencies (mapcat neighbors-memo changed-and-alive))
        live-counts (reduce (fn [acc loc] (assoc acc loc 0)) {} changed-and-alive)
        changed-cells (merge live-counts changed-cells)
        state (assoc state :changed #{})]
    (reduce (fn [acc [loc n]]
              (cond (and (cells loc) (or (< n 2) (> n 3)))
                    (-> acc
                        (update-in [:cells] disj loc)
                        (update-in [:changed] conj loc))
                    (and (nil? (cells loc)) (= n 3))
                    (-> acc
                        (update-in [:cells] conj loc)
                        (update-in [:changed] conj loc))
                    :else acc)) state changed-cells)))

{% endhighlight %}

We have to do a little bit more computation to find the differences, but the benefit is
a huge reduction in cells that we have to update the state of each generation. Let's see
how it performs.

{% include life_sim.html id="third" %}

Almost a threefold increase on 5000 cells! Not too bad at all. It starts slow and speeds
up as the simulation progresses. There might be a way to speed it up even more, and I of
course welcome any improvements. Next time, though, we talk about Gosper's algorithm.

{% include life_includes.html %}
