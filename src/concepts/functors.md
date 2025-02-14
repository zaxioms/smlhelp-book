# Functors
_By Cooper Pierce and Brandon Wu, February 2021_

We have so far discussed the usage of modules for explicitly structuring our
code, in that we are afforded a degree of _modularity_ in how the different
components of some software can fit together. We have used language like
"swapping out" modules or "substituting" modules for one another, but this is
very imprecise. What exactly do we mean when we say to swap one module for
another? Certainly, it would be messy to have to go through our code and change
every mention of `StructureA` for `StructureB`, and we would like to avoid some
kind of global change that requires an extenuous amount of effort on the
client's part.

Going through and changing every mention of a particular structure in order to
achieve "modularization" is akin to saying that using a particular higher-order
function by replacing specifically what each function parameter is in each
invocation is how we can achieve "parameterization" in the function argument. In
reality, this does not offer any more versatility. In this chapter, we will
discuss _functors_, which are akin to _higher-order modules_, which are allowed
to take other modules in as arguments.

## Functors: The Basics

Firstly, consider the signature of a stack.

```sml
signature STACK =
sig
    type 'a t

    val push : 'a t -> 'a -> 'a t
    val pop : 'a t -> 'a option * 'a t
    val size : 'a t -> int
    val empty : 'a t
end

```

Suppose that we would like to _extend_ the definition of a stack such that it
has a limit on how many elements that it contains. However, we don't necessarily
know which implementation of a stack to use - there could be several such
existing implementations, so we would like to instead make our `BoundedStack`
structure a _parameter of_ an existing structure ascribing to `STACK`.

We will then use a very similar signature to `STACK` called `BOUNDED_STACK`,
which is the exact same except for having an `exception Full` for use if a stack
is given too many elements.

```sml
signature BOUNDED_STACK =
sig
    type 'a t
    exception Full

    val push : 'a t -> 'a -> 'a t
    val pop : 'a t -> 'a option * 'a t
    val size : 'a t -> int
    val empty : 'a t
end
```

We first consider the case where we have a hard limit of 10 items in a given
stack.

```sml
functor BoundedStack (S : STACK) :> BOUNDED_STACK =
struct
    type 'a t = 'a S.t

    val limit = 10
    exception Full

    fun push S x =
        if S.size S >= limit
          then raise Full
          else S.push S x

    fun pop S = S.pop S

    fun size S = S.size S

    val empty = S.empty
end
```

We see that most of this code is duplicated - we have based the majority of the
design of this `BoundedStack` in terms of `S`, which is the given implementation
of a stack. The correctness of a `BoundedStack` is thus totally dependent on
whether or not the given `S` is correct, but we achieve _modularity_ in that we
can freely swap out a structure ascribing to `STACK` for another when
instantiating a given `BoundedStack`.

To put it concretely, suppose that we have two structures:

```sml
structure Stack1 :> STACK =
struct
    (* some code here *)
end

structure Stack2 :> STACK =
struct
    (* some code here *)
end
```

Then we can create instances of the `BoundedStack` functor as follows:

```sml
structure BoundedStack1 = BoundedStack(Stack1)
structure BoundedStack2 = BoundedStack(Stack2)
```

**NOTE**: SML functors are _generative_, meaning that applying the same functor
to the same structure twice yields two unique structures. As such,
if we had `structure BS1 = BoundedStack(Stack1)` and `structure BS2 =
BoundedStack(Stack1)`, then the types `BS1.t` and `BS2.t` are recognized as being two
distinct types, despite the fact that they are "constructed" in the same manner.

`BoundedStack1` and `BoundedStack2` implement `BOUNDED_STACK`, so they all can access
the fields of the `BOUNDED_STACK` signature. Presumably, the only change they display
from `Stack1` and `Stack2` are in raising the exception `Full` when the stack is given more than
ten elements.

## Functors: Syntactic Sugar

SML offers some "syntactic sugar" for functor arguments, allowing us to
(seemingly) parameterize them by terms other than structures. It is a little
unsavory to have to hard code the limit of the `BoundedStack` within the functor
itself, rather than having it be parameterized by the limit itself, so we can
actually also write the following:

```sml
functor BoundedStack (structure S : STACK
                      val limit : int) :> BOUNDED_STACK =
struct
    type 'a t = 'a S.t

    exception Full

    fun push S x =
        if S.size S >= limit
          then raise Full
          else S.push S x

    fun pop S = S.pop S

    fun size S = S.size S

    val empty = S.empty
end
```

The only difference is that instead of taking in a single `S : STACK`, we
specify "`structure S : STACK val limit : int`" within the parentheses of the
functor's input. Note that there are no commas or delimiters other than spaces.

In reality, this is something of a lie. While this seems to give the impression
that `BoundedStack` is taking in _two_ things, a structure named `S` ascribing
to `STACK` and a value of type int named `limit`, in reality functors can only
take in other structures. This is thus _syntactic sugar_ for the following code:

```sml
functor BoundedStack (UnnamedStructure :
                      sig
                        structure S : STACK
                        val limit : int
                      end) :> BOUNDED_STACK =
struct
    open UnnamedStructure
    (* same code as before *)
end
```

The `open` keyword specifies to _open_ the namespace within a given module,
effectively promoting all contents of it to the top level. Thus, if we were to
`open Stack1`, as per our previous example, we could write `push` instead of
`Stack1.push`, `pop` instead of `Stack1.pop`, and so on and so forth. Thus, what
this syntactic sugar does is specify to take in a _single structure_ ascribing
to a signature that _contains_ a structure ascribing to `STACK` and an int-typed
value, named `S` and `limit` respectively.

**CAUTION**: This is a very important point to cognize! This can be the source
of many frustrated hours of debugging due to a simple syntax error.

The reason why we must `open UnnamedStructure` in order to be able to use the
same code is because we cannot say `S.push`, for instance, as we did in the
original implementation of `BoundedStack`. We would instead have to specify
`UnnamedStructure.S.push`, which is not what our previous code says. However, if
we `open UnnamedStructure` first, the `S` structure is promoted to the top
level, and we can now access it without first having to go through
`UnnamedStructure`.

The reason for naming the input structure `UnnamedStructure` should hopefully
now be clear. Indeed, it is a _phantom structure_ of a sort, since in the
syntactic sugar case, we never give it a name, and indeed we never really
acknowledge its existence at all. Yet it is important to realize what is really
happening, that there really _is_ a structure being taken in as input, and then
immediately opened for its contents.

What issues can occur if we forget about the existence of this syntactic sugar?
Consider the following code:

```sml
functor BoundedStack (structure S : STACK) :> BOUNDED_STACK =
struct
    type 'a t = 'a S.t

    val limit = 10
    exception Full

    fun push S x =
        if S.size S >= limit
          then raise Full
          else S.push S x

    fun pop S = S.pop S

    fun size S = S.size S

    val empty = S.empty

end

structure BoundedStack1 = BoundedStack(Stack1)
structure BoundedStack2 = BoundedStack(Stack2)
```

This code _will not compile!_ Can you see why?

The issue is that we have added a prefix of `structure` to `S : STACK` within
the inputs. Even though we have only specified one field, not including the
`limit` as a parameter, this will be interpreted as the following:

```sml
functor BoundedStack (UnnamedStructure :
                      sig
                        structure S : STACK
                      end) :> BOUNDED_STACK =
struct
    type 'a t = 'a S.t

    val limit = 10
    exception Full

    fun push S x =
        if S.size S >= limit
          then raise Full
          else S.push S x

    fun pop S = S.pop S

    fun size S = S.size S

    val empty = S.empty

end

structure BoundedStack1 = BoundedStack(Stack1)
structure BoundedStack2 = BoundedStack(Stack2)
```

Thus, `BoundedStack` is no longer a functor taking in a structure ascribing to
`STACK`, but a functor taking in a structure ascribing to `sig structure S:
STACK end`. In other words, taking in a structure _containing_ a structure
ascrbing to `STACK`! Thus, the line `structure BoundedStack1 =
BoundedStack(Stack1)` will not type-check, as `Stack1` does not ascribe to the
same signature that `BoundedStack` is expecting. This one simple syntax error
can be the source of much pain and frustration, so we caution the reader to be
particular with their syntax, and mindful of what is really happening under the
hood.

## Case study: Typeclasses

Certain types have some functionality or operation in common. Depending on the
operation in question, we can say that these types fall into the same
_typeclass_, which is a common interface consisting of a type and the desired
operations. Note that typeclass membership is not a formally defined
relationship, but instead a useful categorization that we use in order to
classify types that we intend to parameterize some implementation over.

For a concrete example of a typeclass, consider the `ORDERED` typeclass.

```sml
signature ORDERED =
sig
    type t
    val compare : t * t -> order
end
```

The `ORDERED` typeclass consists of those types that admit a _sensible
ordering_, which we will not (and perhaps cannot) define. Thus, we can witness
`int` and `string` as valid instances of the `ORDERED` typeclass with the
following structures:

```sml
structure IntOrder : ORDERED =
struct
    type t = int
    val compare = Int.compare
end

structure StringOrder : ORDERED =
struct
    type t = string
    val compare = String.compare
end
```

Note that it is useful, in this case, for our instances of the `ORDERED`
typeclass to be _transparently ascribed_, since it is the whole point that we
are aware of the type that the typeclass is associated with.

**NOTE**: In actuality, we use transparent ascription as somewhat of a
sledgehammer to avoid having to talk about a different language construct,
namely `where` clauses. A `where` clause modifies a signature containing an
abstract type, and concretely specifies what that type should be. For instance,
we could discuss the signature `signature ORDERED type t end where type t =
int`. A structure ascribing to this signature would "publish" the details of the
type `t` (which is really an `int`), the same way that transparent ascription
would. With `where` clauses, however, if there are multiple abstract types, we
can be selective about which ones that are made "transparent". For the purposes
of this chapter, however, we will largely avoid `where` clauses.

These definitions come very naturally from the fact that the `String` and `Int`
libraries included with the Standard ML basis library already implement
`compare` fields, however we can also define types such as `int list` to be
instances of `ORDERED`:

```sml
structure IntListOrder : ORDERED =
struct
    type t = int list
    fun compare [] [] = EQUAL
      | compare [] ys = LESS
      | compare xs [] = GREATER
      | compare (x::xs) (y::ys) =
        case Int.compare (x, y) of
            EQUAL => compare xs ys
          | LESS => LESS
          | GREATER => GREATER
end
```

This structure defines a _lexicographic ordering_ on int lists, using the fact
that values of type `int` are already ordered. It prioritizes the relative
comparison of the corresponding elements of both lists first, and then the
length (akin to how a dictionary is ordered).

Indeed, we can take this one step further and see that lexicographic orderings
form a _functor_, in that we can parameterize the ordering of some type of list,
given that we can order the elements of the list. Like a higher-order function,
this saves us from having to repeat the same code over and over to declare
`StringListOrder` and `CharListOrder` structures, instead encapsulating the
common pattern. We can implement the `LexicListOrder` functor as follows:

```sml
functor LexicListOrder (O : ORDERED) : ORDERED =
struct
    type t = O.t list
    fun compare [] [] = EQUAL
      | compare [] ys = LESS
      | compare xs [] = GREATER
      | compare (x::xs) (y::ys) =
        case O.compare (x, y) of
            EQUAL => compare xs ys
          | LESS => LESS
          | GREATER => GREATER
end
```

Note that since an instantiation of the `LexicListOrder` functor is itself a
structure ascribing to `ORDERED`, it can be "passed in" as input to _itself_,
resulting in _any_ type of nested list being an instance of `ORDERED`, so long
as the base type is also an instance of `ORDERED`.

It is also useful to note that _equality types_ in Standard ML are essentially a
language-supported typeclass, akin to inbuilt support for the following
signature:

```sml
signature EQ =
sig
    type t
    val equal : t * t -> bool
end
```

The operations for `equal` for each "instance" of the typeclass are instead
defined by Standard ML itself, and not user-defined. Thus, we can think of the
equality operator `=` as simply invoking the `T.equal` method for the proper
typeclass `T`, defined by the type that is being compared for equality.

In this next section, we will explore a concrete use for typeclasses when
designing functors.

## Case study: Red-black trees

Typeclasses can be important when we are attempting to place some greater
constraint on the types that may instantiate some universal type. In certain
cases, we do not want the types that we are considering to truly be _any_ type,
but any of a limited subset of types that share some common characteristic or
implement some operation. We will study the use of _red-black trees_ as the
underlying data structure for dictionaries.

A dictionary is a simple data structure that maps keys to values. Consider its
signature given below.

```sml
signature DICT =
sig
    type key
    type 'a dict
    val empty : 'a dict
    val insert : 'a dict -> key * 'a -> 'a dict
    val lookup : 'a dict -> key -> 'a option
end
```

It is a well-known fact that, utilizing a kind of _balanced binary tree_ data
structure, dictionaries can be implemented with an \\( O(\log n) \\) `insert` and
`lookup` operation, as opposed to \\( O(n) \\) for other data structures such as
lists. While there are many different implementations of balanced binary trees,
we will consider a particular variant known as _red-black trees_.

> **[Red-black tree]**: A variant of self-balancing binary tree that ensures
> logarithmic search and insert time. It is named because of its nodes, which
> are marked as either _red_ or _black_. Furthermore, it obeys the following
> properties:
>
> 1. All leaves are black.
> 2. The children of a red node must be black.
> 3. Any path from a given node to a leaf node must go through the same number
>    of black nodes.
>
> Note that as a variant of binary search tree, a red-black tree must also
> satisfy the invariant that the key stored at a node must be greater than or
> equal to every key in the left subtree, and less than or equal to every key in
> the right subtree.

It is easy to reason about why this schema ensures that we have the proper
asymptotic bound for search - the third property in particular ensures that, for
any path from the root, the length of the longest path from the root to a leaf
is at most twice that of the shortest path. This is because the longest such
path you can construct from the root to a leaf (minimizing black nodes) is by
alternating black and red nodes.

This means that a given red-black tree is not as strictly balanced as some other
variants (for instance, AVL trees), however it is always _approximately_
balanced.

We would like to create a structure for red-black tree dictionaries. There are
some options that we have - we could simply hard-code a `TypeRedBlackDict :>
DICT` for any type `Type`, except that this would
entail quite a bit of repeated code (and exertion on our part). Another solution
would be to make the type of `'a dict` doubly-polymorphic instead - something
like an `('a, 'b) dict`, where `'a` is the type of the dict's keys and `'b` the
type of its contents. However, then we lose the guarantee that `'a` is a type
that supports comparison, which means that we cannot satisfy the tree's
invariants.

The solution we will turn to is exactly similar to that as discussed in the
previous section - we will instead design a `RedBlackDict` functor that takes in
a typeclass implementing `ORDERED`, and exports a structure whose keys are the
type of the given typeclass. We thus will define our functor with the following
preliminaries:
```sml
functor RedBlackDict (Key : ORDERED) :> DICT =
struct
    type key = Key.t
    datatype color = Red | Black
    datatype 'a dict = Leaf | Node of 'a dict * (color * (key * 'a)) * 'a dict

    val empty = Leaf
    (* ... *)
end
```

Because we take as input a `Key` structure ascribing to `ORDERED`, we have
access to the `Key.compare` function, which we will use when inserting into our
dictionary. We define a `color` type (which only consists of the constant
constructors `Red` and `Black`) for tagging the nodes of the red-black tree
(leaves are considered to be black).

The question becomes: how should we implement insert? We cannot be so naive as
to simply insert as we would in an ordinary binary search tree, as this would
quickly cause problems with our invariants. In particular, we must be mindful of
the _black height_ invariant, saying that all paths to leaves must have the same
number of black nodes on them.

The easiest case to tackle for insert is the `Leaf` case. How should we finish
the definition of `fun insert Leaf (k, v) = `? Well, clearly we must insert a
`Node(Leaf, (c, (k, v)), Leaf)` for some color `c`. Note that since a `Leaf` is
considered to be colored black, if we choose `c` to be `Black`, we will run into
issues with our black height invariant - we have replaced a `Leaf` (a subtree of
black height 1) with a subtree of black height 2! This will disproportionately
affect the black height of paths ending in this subtree, thus causing the
invariant to be violated.

Thus, the only sensible choice we can commit is to insert as a `Red` node. The
astute reader may see that this will quickly cause issues - we will address this
shortly. Thus, we can write

```sml
fun insert Leaf (k, v) = Node(Leaf, (Red, (k, v)), Leaf)
  | insert (Node (L, (c', (k', v')), R)) (k, v) = ...
```

How should we handle the `Node` case? Well, insertion really only happens at the
leaves - the only thing that we can do at a `Node` is to propagate the change
throughout the tree until it gets to where it needs to be. We have seen that
this schema of an "always-red" insertion maintains the black-height invariant,
however there is the _red-children_ invariant as well - the children of a red
node must themselves be red. This invariant is the one that we are not
respecting, with our current schema.

So we only run into an issue when we insert into the tree such that the new node
is the child of a red node. Furthermore, we know that, if the tree that we are
inserting into is truly a red-black tree, it must respect the red-children
invariant, and thus the the parent of the inserted node must itself have a black
parent. Thus, there can only be four cases for the "site" of the insertion:

<figure class="aligncenter">
    <img src="../assets/redblack.svg" alt="Cases" width="1000"\>
    <figcaption><b>Fig 1.</b> Illustration of the four cases of the red-children
    invariant being broken following insertion. The inserted nodes are marked
    with a "plus". </figcaption>
</figure>

Such an invariant violation is only a local concern, however. All that is needed
in order to restore the invariant is to simple _rotate_ the site of violation,
and do a simple recoloring. We will illustrate only the first case, and the rest
follow similarly. You may verify for yourself that this continues to preserve
the ordering and black-height invariants.

<figure class="aligncenter">
    <img src="../assets/balance.svg" alt="Balance" width="1000"\>
    <figcaption><b>Fig 2.</b> Illustration of the "balancing" necessary in order
    to preserve the red-children invariant in Case 1 of Fig. 1. </figcaption>
</figure>

We thus write the following function which takes care of all four cases.

```sml
fun balance (Node(Node(Node(a,(Red,x),b), (Red,y), c), (Black, z), d)) =
            Node(Node(a,(Black,x),b),  (Red,y), Node(c,(Black,z),d))
  | balance (Node(a,(Black,x), Node(Node(b,(Red,y),c), (Red, z), d))) =
            Node(Node(a,(Black,x),b), (Red,y), Node(c,(Black,z), d))
  | balance (Node(Node(a, (Red,x), Node(b,(Red,y),c)), (Black,z), d)) =
            Node(Node(a,(Black,x),b), (Red, y), Node(c,(Black,z),d))
  | balance (Node(a, (Black,x), Node(b, (Red,y), Node(c,(Red,z), d)))) =
            Node(Node(a,(Black,x),b), (Red,y), Node(c,(Black,z), d))
  | balance T = T
```

Note that if we are not in any of the four described cases, `balance` simply
acts as the identity function, as there is no invariant being broken.

However, this rotation may itself cause another site of red-children invariant
violation, slightly farther up. As such, we must _propagate_ this balancing
operation as far up as necessary, in order to produce a proper binary tree at
the very end. To this end, we can write the following code for the inductive
case of `insert`:

```sml
fun insert Leaf (k, v) = Node(Leaf, (Red, (k, v)), Leaf)
  | insert (Node (L, (c', (k', v')), R)) (k, v) =
    case Key.compare (k, k') of
        LESS => balance(Node(insert L (k, v), (c', (k', v')), R))
      | EQUAL => Node(L, (c', (k, v)), R)
      | GREATER => balance(Node(L, (c', (k', v')), insert R (k ,v)))
```

This code ensures that, after descending into a subtree in order to insert the
given key and value, a balancing operation is immediately performed once the
insertion is complete. This ensures that we have a _bottom-up_ propagation of
balancings, immediately after completing the insertions. Note that because
`balance` acts as the identity function on anything that does not pattern-match
to either of the four cases, we perform only a negligible amount of extra checks
at each recursive call, and ultimately are only concerned with those four cases.

However, this code is not complete. There is a minor edge case that remains -
what if we are too close to the root for any of the four cases to apply? Our
previous analysis relied on the fact that we could assume that the parent of our
inserted node was red, and thus had a black parent - what of the case where the
parent of the inserted node _has no_ parent?

Consider the case illustrated below:

<figure class="aligncenter">
    <img src="../assets/insert.svg" alt="Example of inserting two nodes into an empty tree" width="1000"\>
    <figcaption><b>Fig 3.</b> Demonstration of potential issues in inserting nodes at the root when lacking a black parent. </figcaption>
</figure>

As we can see here, our previous reasoning does not catch this red-children
violation because it does not conform to our previous cases, by virtue of the
inserted node not having a grandfather. This case can _only_ happen at the root,
however, since that is the only location where that can occur. As a result, the
simple solution is to simply make the root of any red-black tree black - it will
preserve the black height invariant, but also result in this red-red violation
being impossible. We can amend our code as follows:

```sml
fun insert' Leaf (k, v) = Node(Leaf, (Red, (k, v)), Leaf)
  | insert' (Node (L, (c', (k', v')), R)) (k, v) =
    case Key.compare (k, k') of
        LESS => balance(Node(insert' L (k, v), (c', (k', v')), R))
      | EQUAL => Node(L, (c', (k, v)), R)
      | GREATER => balance(Node(L, (c', (k', v')), insert' R (k ,v)))

fun insert T (k, v) =
    case insert' T (k, v) of
        Leaf => Leaf
      | Node (L, (_, (k', v')), R) => Node(L, (Black, (k', v')), R)
```

Finally, this results in our completed code for the `insert` function. Note that
because of the signature that we are ascribing to, helper functions such as
`balance` and `insert` will not be visible to the client of the module, so there
is no harm in declaring them within the namespace of the functor.

Our completed code for a red-black tree implementation of dictionaries is thus
as follows. Note that the implementation of `lookup` is very straightforward, and

```sml
functor RedBlackDict (Key : ORDERED) :> DICT =
struct
    type key = Key.t
    datatype color = Red | Black
    datatype 'a dict = Leaf | Node of 'a dict * (color * (key * 'a)) * 'a dict

    val empty = Leaf

    fun balance (Node(Node(Node(a,(Red,x),b), (Red,y), c), (Black, z), d)) =
                Node(Node(a,(Black,x),b),  (Red,y), Node(c,(Black,z),d))
      | balance (Node(a,(Black,x), Node(Node(b,(Red,y),c), (Red, z), d))) =
                Node(Node(a,(Black,x),b), (Red,y), Node(c,(Black,z), d))
      | balance (Node(Node(a, (Red,x), Node(b,(Red,y),c)), (Black,z), d)) =
                Node(Node(a,(Black,x),b), (Red, y), Node(c,(Black,z),d))
      | balance (Node(a, (Black,x), Node(b, (Red,y), Node(c,(Red,z), d)))) =
                Node(Node(a,(Black,x),b), (Red,y), Node(c,(Black,z), d))
      | balance T = T

    fun insert' Leaf (k, v) = Node(Leaf, (Red, (k, v)), Leaf)
      | insert' (Node (L, (c', (k', v')), R)) (k, v) =
        case Key.compare (k, k') of
            LESS => balance(Node(insert' L (k, v), (c', (k', v')), R))
          | EQUAL => Node(L, (c', (k, v)), R)
          | GREATER => balance(Node(L, (c', (k', v')), insert' R (k ,v)))

    fun insert T (k, v) =
        case insert' T (k, v) of
            Leaf => Leaf
          | Node (L, (_, (k', v')), R) => Node(L, (Black, (k', v')), R)

    fun lookup Leaf k = NONE
      | lookup (Node (L, (_, (k', v)), R)) k =
        case Key.compare (k, k') of
            LESS => lookup L k
          | EQUAL => SOME v
          | GREATER => lookup R k
end
```

In the end, usage of modules allows us to write a powerful, parameterized
implementation of a dictionary interface, in such a way that we ensure that our
_representation invariants_ are respected throughout each operation. By making
a structure ascribing to the `ORDERED` typeclass a parameter of the functor
`RedBlackDict`, we allow powerful generality in the type of the key to the
dictionary, without having to introduce additional overhead in the functions of
the module itself.

## Conclusions

In this chapter, we have seen how functors are a potent tool when structuring
our code, that allows us to enforce modularity and implement code reuse _within
the language itself_. Functors also form the basis for a kind of _higher-order
module_, where we can parameterize the structures we are capable of creating by
other structures themselves, resulting in a greater degree of expression and
versatility not unlike those of higher-order functions themselves.
