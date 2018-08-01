# Data modeling in real time collaborative systems

There is plenty of literature available on how to build real-time
systems using
[OT](https://en.wikipedia.org/wiki/Operational_transformation) or
[CRDT](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type)
or
[Diff-Match-Patch](https://code.google.com/archive/p/google-diff-match-patch/).

A real world use of these technologies has a lot of pitfalls and
difficult trade-offs.  A lot of the existing literature tends to focus
on eventual "convergence" of state or issues with the primary
technologies but it seems like not much attention is given to other
important engineering trade-offs involved.

There are two broad (and overlapping) issues:

1. how to effectively model data being mutated by multiple clients.
2. how to deal with user experience issues caused by remote data changes.

## Data integrity constraints

Most collaboration platforms based on OT or similar technology fall
into one of two broad categories -- macro operations or micro
operations.  In the former, the granularity of change is mapped
closely to user actions with roughly one operation corresponding to
one action.  In the latter, the user action is composed of a sequence
of micro-operations where the set of micro-operations is small and
fixed. 

Both approaches exhibit challenges when implementing data-integrity
constraints.

Lets consider an app which uses a graph data-structure
with the hypothetical constraint that there should be no vertices
without any edges.  The user action of deleting a single vertex could
have a cascading effect -- all of its neighbours that were only
connected to it would also need to be removed and so on.

The problem with maintaining this integrity constraint is this: what
happens when two different clients have simultaneous operations that
have "soft-conflicts"  -- one user deletes a vertex, while another
adds an edge?

The edge that is added could "revive" a vertex that would have
otherwise be removed as a side-effect of the first "vertex delete". 

In a collaborative system that uses macro operations, the vertex
delete operation and the edge add operation have a complex
"transformation" procedure.  A clever engineer can no doubt
solve this problem, so this difficulty by itself is not a strike
against that approach.  But when considers how undo-s of these should
be handled, the situation starts to get hairy.

Now imagine that the business requirement changes slightly -- 
say, we now also require a max edges limit on vertices.  The problem
of finding matching transforms is not any harder but the search for
the appropriate transforms often need to start afresh. There seems to
be something of a non-linear nature between requirements and the
transforms -- a small change in requirement sometimes results in a
large change in transform algorithms. Also, often this type of change
would require changing the structure of the operation itself (i.e. the
data sent on the wire along with the operation to perform effective
transforms and merges).  This is particularly the case if one desires
stateless transforms.

    **Aside**: stateless transforms
    
    Stateless transforms are solutions where the act of finding
    compensating actions does not require knowledge of the state of the
    client before the operations. This is a desirable property of
    operations as it allows intermediate servers and tools to be built
    without needing the full potentitally expensive state.  This is
    possible for a lot of simple operations though it is by no means
    easy to devise the transforms to satisfy this property.


When the data structure representing the operation changes, we would
need to consider what happens to the documents with operations for the
previous product scheme. We are left with the choice of having to
rewrite all the operations to match the new world or having to
indefinitely support ("grandfather") both the old and new operations
formats (though a server can reject the new operations for newer
clients).  This adds to the maintenance cost of such a model because
the old and potentially involved code may need to be carried around
for a fair amount of time.

A different maintenance cost arises if new operations on graphs are
devised -- it takes more effort to define the proper transforms for
each of these use cases.

Ultimately, the trade-off with this approach is high-integrity at the
cost of a high on-going maintenance effort.

## Minimal instruction set approach

A collaborative system that uses a minimal instruction set has a
different set of issues.  For the same graph example, one can imagine
modelling this as a `list` of vertex IDs and a sparse adjacency matrix
represented using a `map` whose keys are the `pair of vertices in an
edge` and the value is always `true` (if the pair of vertices do not
have an edge between them, they end up with no entry in the map).

This type of setup is relatively easy to build but it suffers from
very basic integrity constraints -- it is possible to inadvertently
end up with a vertex twice in the list, for example.  This illustrates
one difficulty with the primitive approach -- it takes a little
experience to even spot inconsistencies that can happen.  Lets address
that specific issue by changing the structure of the vertex list to a
map as well -- with keys being the vertex ID and values always being
`true` (just like with the adjacency matrix).  As it turns out, a lot
of the intuition around data-structures most of us have are based on
efficiencies of programming languages (where a vertex list is easier
or more natural to iterate than a hash) but with a micro operation setup,
the choice of basic data-structure needs to be a bit more carefully
thought out.  For example, the vertices collection should really be
thought of as a set.  If we were to restrict the primitives to only
arrays and maps, the set data structure would be most simply expressed
as a map as the add and remove operations work best with maps.

Other problems still exist with the `vertex map and edge map` shared
data structure -- it is possible to have edges referring to vertices
that are not present in the `vertex` list and it most certainly will
allow vertices that don't have any edges. 

One approach to dealing with this issue is to build a `functional`
sanitization algorithm that takes the `vertex list` and `adjacency
map` and returns a sanitized version that meets the product needs.
This would look something like this in ES6/javascript:

```js

// vertices are a map of vertex ID to "true"
// edges are a map of [vertexID1, vertexID2] to "true"
function sanitize(vertices, edges) {
    const newVertices = {}, newEdges = {};
    for (let vid1 in vertices) {
    	for (let vid2 in vertices) {
            const key = normalizeVertexPair(vid1, vid2);
            if (vid1 != vid2 && edges[key]) {
               	newVertices[vid1] = newVertices[vid2] = true;
                newEdges[key] = true;
            }
        }
    }
    const vertexList = [];
    for (let key in newVertices) vertexList.push(key);

    return {
       // return a list of active vertices. an array is easier to work
       // at the higher layers
       vertices: vertexList,
       // provide a check for existence of an edge
       doesEdgeExist(vid1, vid2) {
          return newEdges[normalizeVertexPair[vid1, vid2]] ? true : false;
       },
       // provide a way to iterate through all the edges
       getEdgesOf(vid) {
          const list = [];
          for (let key in newVertices) {
             if (newEdges[normalizeVertexPair(vid, key)]) list.push(vid);
          }
          return list;
      },
      // mutation methos are not provided as each OT system has its
      // own ways of dealing with it.  But the assumption is that
      // mutations will map to changes on [vertices] and [edges] with
      // a sanitize() call afterwards.
   };
}

// returns the key to use into the edge map
function normalizeVertexPair(vid1, vid2) {
    return JSON.stringify(vid1 < vid2 ? [vid1, vid2] : [vid2, vid1]);
}
```

The interesting thing about an approach like this is that constraints
can be changed on the fly relatively easily -- it does not require a
lot of cleverness though it still requires a basic understanding of
all possible states with a given underlying data structure.

There are still problems with changing the constraints -- what happens
to existing documents or models which violate this constraint?  While
this is a product question, the approach allows a degree of latitude
-- only documents that currently violate the constraint are an
issue. Any historically invalid constraints will not cause problems.
Even here, an approach that would work is if we could grandfather
these violations by simply encoding a version flag with the state
itself (this is not dissimilar to the other approach of large
operations)

A compelling benefit of this approach is the fact that developers do
not need to search, implement and test transforms -- instead they
search for the right primitive data-structure, implement the
derivations and test the derivations.  While the search for the right
data-structure still requires a fair degree of cleverness, the
confidence that a data-structure would work is much higher as the
derivation is generally easy to test.  Another side-effect is that
undo works automatically by virtue  of using the primitive instruction
set of already invertible micro-operations. 

But there are still problems with this approach.  The obvious one is
the potential for the data structure to aggregate `turds`.  Like
`non-coding DNA`, there are little bits of data that do not contribute
to the final visible state of the application but take up space.

A general approach to this is `garbage collection` -- all objects
(like vertices) that could be referred from different places can be
stored at a specific "collection" at the root and a general purpose
`purge` algorithm can simply periodically clean up.  This too
has an elegance to it in that if it is implemented once, the ongoing
cost of dealing with this is very low.

A different way of looking at `garbage collection` is canonical
representations.  That is, every document is periodically cleaned up
to its canonical representation (which removes all turds).

A bigger problem with `turds` is unexpected `ghost` state transitions.
Consider a pure input state and one with `turds`, both mapping to the
same derived state.  In most situations like this, there are usually
some operations which when applied to both the input states, will end
up with different derived states.  For example, if the edges list has
references to vertices that have disappeared, if someone adds a new
edge to one of those vertices, the old edges would come back.

Why is this a problem?  For an end user, this would appear to be like
[Hysteresis](https://en.wikipedia.org/wiki/Hysteresis): i.e two
documents with the same appearance will diverge when the same
operation is applied to each. This does not violate the "convergence
of all clients" guarantee as these are two different documents that
have the same appearance. 

But usually, these situations are extremely rare and so an engineering
trade-off can be made in favor of simplicity and easy of change. 

## Functional derived data for integrity constraints

The graph example introduced a specific variety of `derived` data that
can be described in a purely functional fashion based on the raw state
that is used for the collaborative system.  This does pose somewhat of
a performance burden, though this depends on the actual
characteristics of the data (whether these graphs are large and
whether these actions are frequent etc).

Some derivations can actually be specified so that they depend only on
the changes of the input.  For example, a `sum counter` which tracks
the sum of all elements of an array can be implemented like so:

```js

function (oldState, change) {
   if (change.insert) {
       return oldState + change.insert.value;
   } else if (change.remove) {
       return oldState - change.remove.value;
   } else if (change.replace) {
       return oldState + change.replace.newValue - change.replace.oldValue;
   }
}
```

If the original raw data is never directly accessed by the application
(as it was with the graph example), these type of solutions do not
ever even have to store the underlying state at all and can simply
store the derived state only.  This may seem contrived but the
`counter` above is actually an elegant way to implement
`increment/decrement` counters in a collaborative system which
supports only `lists` and `maps`.  The actual operation of increment
or decrement can be represented as a change which inserts the
increment or decrement into the underlying array and if the only state
ever maintained is the derived state of the `sum` as described above,
the storage requirements are fixed rather than ever-growing.

Such incremental calculations perform very well even if the underlying
state is needed for other purposes.  But most collaborative systems
only guarantee that the underlying state converges -- to be sure that
the derivation also converges, a raw functional specification based
only on the underlying state is still required (at least for
verification in the test suite).

Not all derivations can be expressed as incremental computations. The
graph example above is actually one of those that **cannot** be easily
represented as an incremental computation.  At a minimum this implies
duplication of state (which is probably not a huge issue for most
situations) but in the worst case, it would also be a performance
issue.  But there are usually simple optimizations possible, such as
implementing a hybrid solution that optimizes the calculation by
looking at both the change and the input state. Such solutions suffer
from the risk of bugs (as proving their correctness is not as easy as
it is with functional or incremental code with well understood input
state spaces).

A more interesting question is what types of constraint problems are
even covered by purely functional derivations, so let us consider a
couple of other problems

### King of the Hill

[Terry Crowley]
(https://hackernoon.com/real-time-editing-is-kind-of-hard-785622043b7b)
talks about a particular problem implementing a game of `King of the
Hill`.  Here is a paraphrase of the constraints:

1. There can be at-most one player who is a King of the Hill.
2. There can be any number of non-King players
3. A player cannot be both King and non-King.
4. If there is at least one player, one of them has to be King.

The particular physical data structure used by
[Crowley](https://hackernoon.com/@terrycrowley) in that article is very
likely chosen to illustrate that it is tricky to find the right data
structure or transform, so it is not fair to compare any
representation here against that.  The motivation here is whether
there is a way to look at this from the point of view of derived data
structures that simplifies the problem.

Here is one simple solution for it:

```json
{
   "players": {
     "playerID": {
         "Name": "PlayerName",
         "LastKingOfHillTimeCrownedAt": "time",
      }
   }
}
```

Here, the players are stored as a map based on their ID.  Whether they
are king of the hill or not is described by a attribute on the player
structure which manages the time when they were last made king of the
hill.  The use of a time field is to solve the `delete` problem that
`Terry` talks about (he uses a more technically accurate solution but
this is acceptable for the purposes of this discussion).

So, the actual king of the hill requires a computation of finding the
player with the largest `King Of The Hill` time.  This is a `pure
functional` derivation that also happens to be quite easily
implemented as a **mostly** incremental operation (when a player gets
deleted, a re-computation is needed).

How easy is it to come up with a solution like this? It turns out that
this pattern (of tracking the last) is not that unnatural if we start
thinking of all actions in terms of the state.  If there is a
requirement that removing a KingOfTheHill player requires election of
another KingOfTheHill, the effective requirement from the app is that
for consistency reasons, all clients should be able to compute the
next KingOfTheKill purely from the state alone.  So, storing the
"time" helps with this.  Note that rephrasing this problem in terms of
the state immediately suggests a simplification: instead of time, an
incrementing counter would work as well (with a simple disambiguation
needed if there are two simultaneous updates with the same counter)

A reasonable engineering objection to the adding of the
`LastKingOfHillTimeCrownedAt` field to the player structure is that
this violates [separation of concerns]
(https://en.wikipedia.org/wiki/Separation_of_concerns).  In an ideal
world, this data structure should be managed separately.  One can
easily consider storing a separate map like so: 


```json
{
   "players": ...,
   "king-of-hill-time": {
        "player-id": "time",
    }
}
```

This has a little bit of a downside.  Whenever a player gets deleted,
either the app needs to know to delete this map (which would violate
the separation of concerns) or whenever the derivation is being done
for the king-of-the-hill, the player collection must be checked to see
if the player exists.  The latter solution is relatively
straight-forward and probably preferable but it has the downside that
the map of `king-of-hill-time` will have `turds` of deleted
players. A general garbage collection process can probably be
implemented if all the object collections were maintained as top-level
objects and all references used IDs from top-level object
collections.  This is actually relatively easy to code and maintain
and the performance depends on a variety of factors.

A different argument can be made about separation of concerns: if the
`king-of-the-hill` has a life-cycle dependency on `player`, it is
probably the right data structure to stick this as a field of the
`player`.  The problem of working with an effectively global namespace
can be dealt with elegantly if the underlying system (both the
platform and the language) supported namespaces for these attributes.
This is obviously a lot of engineering effort that is of questionable
value for anything but very large projects.  In a lot of ways, this
way of thinking about data in terms of life-cycle and namespaced
attributes is similar to the programming language concept of
[Traits](https://en.wikipedia.org/wiki/Trait_(computer_programming)).

### Unique user-visible names

Consider a game where people chose their tags.  The problem here is to
guarantee uniqueness of gamer tags.  Conflicts are inevitable and easy
product solutions exist (such as tacking on unique numbers etc) but
lets consider a demand for a specific scheme: if multiple players
choose `xyz` simultaneously, only one of them gets the `xyz` while the
rest get `xyz #1`, `xyz #2` etc.  If a player with one of these `xyz`
tag is deleted, no existing player's tag will get reassigned.  But if
a new player comes in requesting `xyz`, he will get `xyz` (or the
lowest available tag slot).

This is obviously somewhat contrived but lets specify the rules in a
bit more detail:

1. Players can choose any tag such as `xyz` (or even `xyz #42` but
then the `#42` is a hint, no requirement to honor it).
2. The system assigns a tag that is either what the player requested
(preferable) or of the scheme `xyz #nnn`
3. When players leave or arrive, it does not affect the tag of other
players.
4. When a player gets a name they didn't request, no lower slot name
is available (i.e. if they get `xyz #3`, `xyz #2` must be currently
taken).

This pathological example maybe a case of a badly-formed requirements
as one can easily adjust the requirements (such as dropping the lowest
available slot) to make this relatively simple.  But as the
requirements stand, this is extremely difficult to describe in a pure
functional way.

A brute force solution is to maintain a sequence of events that affect
a given tag.  That is, for each tag, we maintain an array with `join`
and `leave` entries. The actual set of gamer tags associated with
different participants is always calculated by iterating over the
array in order.  Note that given the assignments at that point and a
`join` or `leave` event, we can reliably calculate the next
assignments, so in theory the data structure can just be an initial
assignment map and recent events (instead of storing all the events).
This is similar to garbage collection.

### General approach

A pattern might be seen with the general approach:

1. Try to capture all the information needed to describe the behavior
of the system directly in the state.
2. Do not optimize for data structures too soon because they may make
the set of valid states too large and cause merge conflicts.
3. Instead unroll as much of the state into flat structures as
possible where the consistency guarantees of arrays and sets are all
that are used.
4. Describe the required derived structure as a function expression of
all possible states.
5. Finally, look at storage and performance optimizations using
garbage collections and incremental calculations that are equivalent.

### Cursor problem

Another common problem with collaboration systems is semi-derived
data. A common example of this is the `collaborative cursor` in
`collaborative document editing` apps.  Other variations of this problem crop
up with non-document like apps too (such as shared list editing or
shared spreadsheets or even chat and facebook).

The natural way to model the current location a user is within a
document is by considering this as a separate state `{start,
end}.`  A naive way of storing start would be by index into the
document (assuming it is some form of linear text) but this breaks
down when we consider what needs to happen when an insert happens
before this text -- the start/end need to be transformed based on the
changes that happened.

It is relatively easy to write some code that maintains a start/end
based on changes to the text:


```js
function updateCursor({start, end}, change) {
    // if change happend before start ... etc
    return {start: newStart, end: newEnd};
}
```

But this is a non-functional description (because it depends on the
change rather than the current state of the document) and as such
there is no guarantee that the cursor of a particular user will
converge to the same place for every user.

The issue is not the importance of that.  For a collaborative text
editor, it is the right engineering choice to not worry about that
divergence since this is ephemeral data.  But for a game, this may
change how points are calculated etc.

The issue is that it is not immediately obvious that the naive way of
doing this may cause the derivation to diverge.

This particular problem occurs frequently enough that the OT platform
underneath can provide the transforms (which are basically regular
array transforms) that guarantee convergence (since they are also used
for regular OT transforms themselves).  This is the intended approach
in the [DOT](https://github.com/dotchain) project but most OT systems
simply sidestep the issue.

Another problem happens when one considers what happens if the cursor
is not stored with the model at all but maintained and communicated
separately.  This is, in fact, how most collaborative systems seem to
be operating today.  The issue here is that the `start/end` offsets
only make sense with a particular version of the document and clients
may receive offsets well before they are synchronized up to that
version of the underlying document.  So, a client which inserted a
large chunk of text at the end of the document might send its current
cursor (at the end of the document) and this cursor update may be
received by clients before the update to the underlying document.
If the receiving client attempted to use the new cursor position, it
would get out-of-bounds exceptions (though it can obviously consider a
dynamic derivation that limits a peer's cursor to be within its
current documentation).  But more severe divergences can happen in
this case (and some common collaborative editors actually exhibit
these issues).

A different way of looking at this particular problem eases this pain
considerably -- by looking at collaborative cursors as effectively
part of the document itself.  This guarantees the order of the
operation but also removes any need for the client app to worry about
transformations.  This has other problems though (a cursor move now
appears to be an edit of the document) that it might be a case of
where the cure is worse than the disease for some applications.

### References

The general recommendation is to build the idea of references as part
of the low level collaboration system to enable higher level apps to
be written with ease and confidence.  Note that real world
applications need references between models (or documents) and there
are two kinds of references conceptually: the `live` version is
specified with a basis to a particular version of the document and it
is maintained by the collaborative system answering the question of
where that reference is with a later (or even earlier) version of the
document.

A second version is the effectively static version where if someone
refers to the `top` player, he is always referring to the array
element indexed at zero.

### Second order derivations

Consider derivations based on derived data -- for example, consider
the earlier graph example.  Now lets assume we want to keep count of
number of partitions of the graph (i.e. no edges between partitions)
exist.  This is a simple derivation and can be functionally
expressed.  But the ability to do incremental derivations on this is
lost.

One approach is if the data structure used to model the derived system
is itself using a "shared" model with only one writer.  This feels odd
but the benefit of that approach is there is now a unique programming
interface for making changes to data structures and secondary
derivations start looking similar to primary derivations.  They follow
the same rules of convergence:

   A derivation that is fully expressible as a pure function of only
   the state of the inputs is guaranteed to converge.

   Any incremental derivation that implements an optimization of such
   a pure function can be tested relatively easily by comparing
   against the pure function.


But there are execution order issues similar to references.  Consider
a functional derivation that depends on two other derivations. In the
example below, the last function `percentageAboveOrAverage` attempts
to calculate how many elements are at average or above average.

```js
function mean(arr) {
     let sum = 0;
     for (let kk = 0; kk < arr.length; kk ++) sum += arr[kk];
     return sum/kk;
}

function averageOrAbove(arr, mean) {
     let result = [];
     for (let kk = 0; kk < arr.length; kk ++) {
         if (arr[kk] > mean) result.push(arr[kk]);
     }
     return result;
}

function percentageAboveOrAverage(arr, avgOrAbove) {
     if (arr.length === 0) return 0;
     return 100 * arr.length / avgOrAbove.length;
}
```

While the `pure functional` aspect of all the functions guarantees
that the final derivation will converge to the same value irrespective
of what history the input derivations have, it does not guarantee that
intermediate calculations will not crash. For example, if the input
array changes from empty to one element but the first calculation done
is `percentageAboveOrAverage` (before the other two functions are
scheduled), the method would return an invalid result.  Note however,
that when the other two functions change and their result is then fed
back into the `percentageAboveOrAverage` function, it will properly
converge to the right value.

The example above is a bit contrived -- but if all the individual
functions were incremental functions depending on changes on their
underlying values, it gets difficult to ensure that the change events
arrive in the right order.

This leads to a strong requirement that derivations are executed up in
the right order.  This would start looking like the startup and
dependency hell that most engineers are familiar with.

    Aside: FRP
    The order-of-execution problem would be familiar to anyone who has
    worked with functional reactive programming.  It is not unique to
    collaboration but working with events and changes just makes this
    a more frequent occurence.


### Pull request model, manual intervention

A completely different way to deal with constraints is via a flow
similar to github's PRs.  This approach is feasible with OT-based
collaborations as the underlying technology is analogous to git.

One implementation here is for clients to make changes to their own
branches at all times and sending up the changes to the server as a
"Pull request".  The server merges them into master if there are no
constraint violations (after the full chain of changes from the
client). If there are constraint violations, it is reported to the
client (though the client can probably keep track of this itself based
on the current state of the master).

The client uses the same technique to merge master into its branch
periodically.  If there are conflicts, it would need to update its
branch to resolve the conflicts before it can merge up to master.

This approach involves quite a bit of OT infrastructure underneath but
exposes a very elegant user-involved conflict resolution for cases
where that is the right thing to do.  It is also an useful approach to
let users make changes before they commit to them.

## User experience jank

The second part of the equation is user experience issues when working
with data that is changing underneath by edits not made by the current
user.

### Cursor preservation

[Neil Fraser](https://neil.fraser.name) talks about [this
issue](https://neil.fraser.name/writing/cursor/) with OT systems. The
underlying issue is similar to the [References](#references) issue
discussed earlier.  Basically when a user is editing a document, edits
from other users can affect one's cursor position making the
experience feel janky.

But another part of the equation is that edits from other users can
cause the screen contents to move in unexpected ways.  Consider a user
on the fourth line of a text.  If a remote user inserts a thousand
lines at the start, the cursor may now end up off-screen.

This is very similar to integrity constraints and one can describe the
problem in similar terms:

    The physical position of the current cursor of a user shall
    move the least amount possible in case of edits from other users.


There are several wrinkles on it.  Consider if the user has scrolled
the screen manually up to look at a different part of the document.
In this case, we want to preserve whatever it is the user has his/her
attention on. For office or word documents, there are several
candidates: section headers, tables, first visible line etc.  A good
basic compromise is the first visible line (with the exceptions around
start of document and end of document -- if either are visible then
the logical start and logical end position are what is generally
expected to be preserved).

### Scroll jank

A different kind of "jank" experience is when a user is scrolling a
region and there are changes that happen meanwhile.  While it is
possible to maintain scroll position correctly, the issue is one of
performance. For example, updating a DOM to load a new image can cause
jitter with an ongoing scroll.

A simple workaround here is to suspend synchronization while a scroll
is in progress.

### Drag and drop jank

A rare but deeply frustrating jank happens when a user is about to
drag something or about to drop something and the screen underneath it
changes in some way.

The workaround for this is to suspend synchronization when mouse is
down and re-enabling it a few seconds after the mouse goes up (to
separate the animation of drop from any side-effects from the merge).

### Input jank

For rich applications, another source of jank is within a control or
dialog and a remote change invalidates the whole control/dialog.  A
naive implementation would simply reset the cursor elsewhere.  But
there is another rule, violations of which are extremely frustrating
to users:

    When a user is typing, the focused element should never change
    without user consent as a lot of users type fast and a shift in
    focus will cause unexpeted input to be entered.

A weaker version of it applies to content that is only being viewed.

Implementing this is tough -- a brute-force solution is to identify
when a user enters these "unsafe" UX experiences and simply freeze
collaboration. 

A different approach is to have the system provide locking/transaction
semantics.  A lock does not prevent edits from other users but it
prevents the local clients from seeing changes to locked regions of
data.  This is a more granular version of the brute force
approach. The risk with this approach is the complexity of deferring
changes to one part of the state which is effectively reordering
operations as well as the possibility of breaking integrity
constraints when such reordering happens.

### View jank

A different kind of jank happens when someone changes the view of an
interactive element.  Consider a graphics editor with many layers.  If
someone is drawing on one layer and a different user reorders the
layers, this can cause some serious issues.  A similar kind of issue
happens when someone is editing a bar-chart and a different user
swaps the x and y axis.

An app that naturally supports per-client views of data or a
pull-request model for data are less likely to suffer from these
issues.

## Conclusion

Derived data is not easy to work with. Most real-world collaborative
applications can benefit from a richer framework for derived data.

The collaboration platform should not stop at convergence of raw data.
The platform effectiveness will be drastically improved if there is
built in support for common derived data situations, including the
managing of derived data dependencies.

UX jank increases with degree of collaboration and require a variety
of techniques to address.  Advanced platform help is useful in a lot
of cases.
