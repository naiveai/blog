---
title: "Disjoint Sets in Rust"
datePublished: Sun Apr 02 2023 19:36:35 GMT+0000 (Coordinated Universal Time)
cuid: clfzszrk900050amqdduk237h
slug: disjoint-sets-in-rust
tags: algorithms, data-structures, rust

---

We have a set of villages on a map. And at first, which means each village is isolated from other villages.

1. If village A is connected to village B, then village B is also connected to village A. That means a connection is **symmetric**.
    
2. If village A is connected to village B, and village B is connected to village c, then we say that village A is also connected to village C. That means a connection is **transitive**.
    

Each time, we can build a road to connect any two villages if they are not connected. Based on this situation, it comes with a few problems to be resolved:

We want to know how many disjoint components exist after some connecting operations. We want to know which village belongs to which disjoint component efficiently. Given any two villages, we need to know if they are connected. Given any two villages, we need to be able to connect them if they are not connected.

So this is the interface we need, represented in a Rust trait:

```rust
trait DisjointSet<T> {
    type ElementIdentifier;

    /// Creates a new subset with a single element `elem`. If the operation
    /// succeeded, it returns a Some with a unique identifier that can be used
    /// in future calls to the disjoint set in order to operate on the new
    /// subset.
    fn make_subset(&mut self, elem: T) -> Option<Self::ElementIdentifier>;

    /// Checks if two elements are in the same set, returning Some(true) if they
    /// are, Some(false) if they aren't, or None if either isn't present.
    fn same_set(
        &self,
        elem_a: Self::ElementIdentifier,
        elem_b: Self::ElementIdentifier,
    ) -> Option<bool>;

    /// Merges the subsets containing the two elements so that `same_set(A, C)
    /// == true` both if it already was, or if before the union `same_set(B, C)
    /// == true`, and vice versa. I.E. {A} ∪ {B, C} == {A, C} ∪ B == {A, C}
    /// ∪ B. Returns Some(true) if the operation was performed, Some(false) if
    /// they were already in the same set, and None if either doesn't exist.
    fn union(
        &mut self,
        elem_a: Self::ElementIdentifier,
        elem_b: Self::ElementIdentifier,
    ) -> Option<bool>;
}
```

The seemingly obvious implementation is a 2D array:

```rust
#[derive(Debug, PartialEq, Eq)]
struct NaiveDisjointSet<T: Eq + Hash> {
    sets: Vec<HashSet<T>>,
}

impl<T: Eq + Hash + Clone> DisjointSet<T> for NaiveDisjointSet<T> {
    type ElementIdentifier = usize;

    fn make_subset(&mut self, elem: T) -> Option<Self::ElementIdentifier> {
        let elem_set_idx = self.sets.len();

        self.sets.push(HashSet::from([elem]));

        Some(elem_set_idx)
    }

    fn same_set(
        &self,
        elem_a_set_idx: Self::ElementIdentifier,
        elem_b_set_idx: Self::ElementIdentifier,
    ) -> Option<bool> {
        Some(elem_a_set_idx == elem_b_set_idx)
    }

    fn union(
        &mut self,
        elem_a_set_idx: Self::ElementIdentifier,
        elem_b_set_idx: Self::ElementIdentifier,
    ) -> Option<bool> {
        let set_a = self.sets.remove(elem_a_set_idx);
        let set_b = self.sets.remove(elem_b_set_idx);

        self.sets.push(set_a.union(&set_b).cloned().collect());

        Some(true)
    }
}
```

But this has some problems. Mainly, the issue is that the `union` operation is really expensive both in terms of time and memory: we have to perform a O(n) `HashSet` union operation on potentially larger and larger N's as more and more village sets get merged together. And there's another problem that Rust makes more obvious: the need for `clone` and `collect`, which are also O(n) operations themselves. This also means that the types have a `T: Hash + Clone` restriction, which also is especially inelegant.

More importantly than even any of those, in this case the *caller* is responsible for keeping track of the internals of which element is in which set index, otherwise they're not able to perform operations on them. And while we could make the internals more complicated to account for that, long story short: it is quite difficult to do and ultimately still results in internals being exposed in awkward ways.

There is a better way, and while it may seem more complex on the surface, it is in many ways remarkably elegant: the disjoint set forest.

```rust
/// A common implementation of a disjoint set involving a forest.
struct DisjointSetForest<T: PartialEq> {
    roots: HashSet<usize>,
    nodes: Vec<Node>,
    elems: Vec<T>,
}

struct Node {
    rank: usize,
    parent: Cell<usize>,
}

impl<T: PartialEq> DisjointSet<T> for DisjointSetForest<T> {
    type ElementIdentifier = usize;

    fn make_subset(&mut self, elem: T) -> Option<Self::ElementIdentifier> {
        if self.elems.contains(&elem) {
            return None;
        }

        // This is the index where the element will be inserted, thanks to
        // 0-indexing. We need to get this before we actually push the element.
        let element_idx = self.elems.len();

        self.elems.push(elem);

        self.nodes.push(Node {
            rank: 0,
            parent: Cell::new(element_idx),
        });

        self.roots.insert(element_idx);

        Some(element_idx)
    }

    fn same_set(
        &self,
        elem_a: Self::ElementIdentifier,
        elem_b: Self::ElementIdentifier,
    ) -> Option<bool> {
        Some(self.find_root(elem_a)? == self.find_root(elem_b)?)
    }

    fn union(
        &mut self,
        elem_a: Self::ElementIdentifier,
        elem_b: Self::ElementIdentifier,
    ) -> Option<bool> {
        let (mut a_root_idx, mut b_root_idx) = (self.find_root(elem_a)?, self.find_root(elem_b)?);

        // In this case, they're already in the same set. We don't use same_set
        // to check this because if it's not true, then we perform redundant
        // work since we need the representatives for future work.
        if a_root_idx == b_root_idx {
            return Some(false);
        }

        if self.nodes[a_root_idx].rank < self.nodes[b_root_idx].rank {
            // We want the element we call A to always be higher rank.
            self.nodes.swap(a_root_idx, b_root_idx);
            mem::swap(&mut a_root_idx, &mut b_root_idx);
        }

        // Now a_root.rank >= b_root.rank no matter what.
        // Therefore, make A the parent of B.
        self.nodes[b_root_idx].parent.set(a_root_idx);
        self.roots.remove(&b_root_idx);
        if self.nodes[a_root_idx].rank == self.nodes[b_root_idx].rank {
            self.nodes[a_root_idx].rank += 1;
        }

        Some(true)
    }
}

impl<T: PartialEq> Default for DisjointSetForest<T> {
    fn default() -> Self {
        Self {
            roots: HashSet::new(),
            nodes: vec![],
            elems: vec![],
        }
    }
}

impl<T: PartialEq> DisjointSetForest<T> {
    fn new() -> Self {
        Self::default()
    }

    fn find_root(
        &self,
        elem: <Self as DisjointSet<T>>::ElementIdentifier,
    ) -> Option<<Self as DisjointSet<T>>::ElementIdentifier> {
        if self.roots.contains(&elem) {
            return Some(elem);
        }

        let mut current_idx = elem;
        let mut current = self.nodes.get(current_idx)?;

        while current.parent.get() != current_idx {
            let parent_idx = current.parent.get();
            let parent = &self.nodes[parent_idx];

            // Set the current node's parent to its grandparent. This is called
            // *path splitting*: (see the Wikipedia page for details) a simpler
            // to implement, one-pass version of path compression that also
            // turns out to be more efficient in practice.
            current.parent.set(parent.parent.get());

            // Move up a level
            current_idx = parent_idx;
            current = parent;
        }

        Some(current_idx)
    }
}
```

This results in a forest of trees that make themselves simpler to look through every time they're looked through, which means that the gains add up extremely fast in a large operation with many sets and many union operations. It's super cool!

If you want to read more about this, as cliche as it sounds, the Wikipedia article on union-find is actually a really good explanation once you understand what the goal is, which you hopefully do now. If you liked this, please do share your thoughts on it in the comments below!