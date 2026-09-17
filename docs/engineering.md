# Engineering notes — Missile Simulation

Missile Simulation is a small Java Swing game built around weighted graphs and Dijkstra's shortest-path algorithm.

The interesting part is not the theme. It is the translation between three representations of the same problem:

```text
weighted graph
    │
    ├── rendered as a map image
    ├── exposed as legal button choices
    └── encoded as an adjacency matrix for Dijkstra
```

This document describes what the 2022 implementation actually does, where its assumptions live, and what a cleaner version would look like today.

---

## 1. Runtime flow

The application entry point is `PDSA_CW2_T1`.

```text
PDSA_CW2_T1.main()
        │
        ▼
      Home
        │
        ▼
   Instruction
        │
        ▼
      Map1
        │
        ├── success -> MBox1 -> Map2
        └── failure -> MBoxFail -> retry Map1 / home
                         │
                         ▼
      Map2 <-------------┘
        │
        ├── success -> MBox1 -> Map3
        └── failure -> MBoxFail -> retry Map2 / home
                         │
                         ▼
      Map3 <-------------┘
        │
        ├── success -> MBox1 -> Last_Page
        └── failure -> MBoxFail -> retry Map3 / home
```

`MBox1` is the successful-level router. `MBoxFail` handles failure recovery.

The original project always sent **Retry** back to Map 1. The cleaned version now uses the active `MapNum` marker so a failed Map 2 or Map 3 attempt retries that same level.

## 2. The graphs

Every level has an undirected weighted graph encoded as an adjacency matrix.

A `0` means “no edge” in the project. That means this representation cannot express a real zero-weight edge.

### Level 1

```text
    A  B  C  D
A   0  3  6  0
B   3  0  2  8
C   6  2  0  5
D   0  8  5  0
```

Target: `D`

Candidate routes include:

```text
A -> B -> D       3 + 8     = 11
A -> C -> D       6 + 5     = 11
A -> B -> C -> D  3 + 2 + 5 = 10
```

Shortest distance: `10`.

### Level 2

```text
    A  B  C  D  E
A   0  7  5  0  0
B   7  0  3  4  6
C   5  3  0  4  0
D   0  4  4  0  6
E   0  6  0  6  0
```

Target: `E`

Useful route comparisons:

```text
A -> B -> E       7 + 6     = 13
A -> C -> B -> E  5 + 3 + 6 = 14
A -> C -> D -> E  5 + 4 + 6 = 15
A -> B -> D -> E  7 + 4 + 6 = 17
```

Shortest distance: `13`.

### Level 3

```text
    A   B   C   D   E   F
A   0  10  15   0   0   0
B  10   0   0  12   0   5
C  15   0   0   0  10   0
D   0  12   0   0   2   5
E   0   0  10   2   0   5
F   0   5   0   5   5   0
```

Target: `F`

Useful route comparisons:

```text
A -> B -> F            10 + 5          = 15
A -> B -> D -> F       10 + 12 + 5     = 27
A -> C -> E -> F       15 + 10 + 5     = 30
A -> B -> D -> E -> F  10 + 12 + 2 + 5 = 29
```

Shortest distance: `15`.

## 3. Player route state

The current game does not store the selected route as a `List<Node>`.

Instead each map keeps a handful of integer flags and a running total:

```text
distancecount
b
c
d
e
r
```

The flags mean that a node has already been visited or that a special state has been entered.

A button click does three things:

```text
1. add the appropriate edge weight to distancecount
2. disable nodes that should no longer be selected
3. enable nodes that are valid next steps
```

For example, on Map 1:

```text
A -> B adds 3
B -> C adds 2
C -> D adds 5
```

which makes the winning total `10`.

The UI therefore acts as both the route input and a lightweight state machine.

## 4. Dijkstra as implemented

Every map contains its own copy of the shortest-path code.

At launch time the algorithm roughly does this:

```text
for each matrix entry:
    if edge weight == 0:
        replace with 999

copy start-node row into distance[]
mark start visited
set start distance to 0

repeat:
    find unvisited node with minimum tentative distance
    mark it visited

    for every unvisited neighbor:
        if distance-through-current is shorter:
            update its tentative distance
```

This is the classic array-based form of Dijkstra.

There is no binary heap / priority queue. Selecting the next node scans the whole `distance` array each time, giving approximately:

```text
O(V²)
```

For `V <= 6`, that is completely adequate.

## 5. What `999` means

The graph matrices use `0` to mean “not connected.”

Before traversal, the code replaces every zero with `999`.

That creates a simple infinity sentinel:

```text
0 in matrix -> no edge
999 at runtime -> effectively unreachable
```

This works because all real edge weights are tiny compared with 999.

It does have two constraints:

- a valid zero-cost edge cannot be represented;
- the sentinel is an implementation convention rather than a true infinity value.

A cleaner implementation would leave the source matrix immutable and use `Integer.MAX_VALUE`-style distance initialization instead of rewriting the matrix.

## 6. The target node is implicit

The algorithm computes distances to every node, but the current code does not name a target explicitly.

After Dijkstra completes it runs:

```java
for (int i = 0; i < nodeCount; i++) {
    shortestdis = distance[i];
}
```

Because each level's target is deliberately the final node in the matrix—`D`, `E`, then `F`—the last assignment is the distance the game wants.

So the real hidden invariant is:

```text
target index == graph.length - 1
```

That works for these three levels, but it is brittle. If a future level targeted node `C`, the algorithm would still judge the distance to the last array entry.

A modern solver should accept the target explicitly:

```java
shortestDistance(graph, startIndex, targetIndex)
```

## 7. The `pred` array is unused

Each map allocates:

```java
int pred[] = new int[nodeCount];
```

and initializes it, but never records predecessors during relaxation.

As a result, the algorithm calculates only distances. It cannot reconstruct:

```text
A -> B -> C -> D
```

as the optimal path.

That is why the success condition compares only numeric cost:

```text
player distance == shortest distance
```

For this game that is enough, but a more complete Dijkstra implementation would update a predecessor array and reconstruct the path after traversal.

## 8. Distance equality versus path equality

The game validates cost, not node sequence.

Conceptually:

```text
player route cost == optimal route cost -> success
```

It does **not** check:

```text
player nodes == reconstructed Dijkstra nodes
```

That means two different legal paths with the same optimal total would both be accepted.

For a shortest-path game that is arguably the correct behavior: multiple shortest paths can be valid answers.

The important limitation is that the current solver cannot tell the player *which* route Dijkstra found because it does not preserve predecessors.

## 9. Two sources of truth for edge weights

This is the most important structural weakness in the original implementation.

Each edge weight exists twice.

Once in the adjacency matrix:

```java
{0, 3, 6, 0}
```

and again inside button handlers:

```java
distancecount = distancecount + 3;
```

So graph definition and player-route scoring can drift apart.

If someone changed the matrix from `A-B = 3` to `A-B = 4` but forgot the button handler, the rendered game could reject a route based on inconsistent data.

The graph should be the single source of truth.

A cleaner route action would be:

```java
routeDistance += graph[currentNode][nextNode];
currentNode = nextNode;
```

No edge number would be duplicated in event code.

## 10. Three map classes versus one level renderer

`Map1`, `Map2` and `Map3` are separate Swing frames with very similar responsibilities:

```text
render graph art
render node buttons
track current path
calculate shortest distance
judge launch
reset state
```

The differences are mostly:

- adjacency matrix;
- node count;
- map image;
- available edges;
- layout/button count.

A data-driven rewrite would use one reusable game screen.

```java
final class LevelDefinition {
    int[][] graph;
    String[] nodeLabels;
    int start;
    int target;
    String artwork;
}
```

Then:

```text
GameScreen(LevelDefinition map1)
GameScreen(LevelDefinition map2)
GameScreen(LevelDefinition map3)
```

That would eliminate most of the duplicated algorithm and reset code.

## 11. Static state

The level screens use static fields such as:

```text
MapNum
distancecount
b
c
d
e
r
```

Static state makes it easy for small Swing screens to communicate without passing objects around, which is likely why it felt convenient in the original coursework version.

The trade-off is hidden coupling.

For example, `MBox1` knows which screen to open by reading static fields owned by three other windows.

A cleaner design would have one explicit game session:

```java
GameSession
├── currentLevel
├── currentNode
├── route
└── routeDistance
```

The UI would receive that session rather than discovering state through unrelated classes.

## 12. Success navigation

`MBox1` controls progression after a successful launch.

Its logic is effectively:

```text
Map1 completed -> Map2
Map2 completed -> Map3
Map3 completed -> Last_Page
```

The map classes communicate completion by setting their static `MapNum` marker before opening the success window.

This is small enough to work, but it mixes level state with navigation state.

In a larger application, the current level should be represented once and the navigator should switch on that value.

## 13. Failure and retry

The original failure screen contained:

```text
Retry -> Map1
```

regardless of which level failed.

That meant failing Map 3 threw away the player's progress through Maps 1 and 2.

The cleaned repository now checks the active map marker and reopens the level that actually failed.

Returning to the main menu also clears all three map markers so a fresh run starts with clean navigation state.

This is the only gameplay behavior intentionally changed during the repository overhaul.

## 14. Reset behavior inside a level

Each map includes a one-use `Reset(1)` button.

Reset does not reload the whole Swing frame. It manually restores the initial node/button state and sets the accumulated route distance back to zero.

That is another place where a session model would help.

Current style:

```text
set b/c/d/e flags to 0
set distancecount to 0
restore button colors
restore enabled/disabled state
```

Data-driven style:

```text
session.reset()
render(session)
```

The latter keeps domain reset logic independent from Swing components.

## 15. Swing and the Event Dispatch Thread

The generated `main` methods use:

```java
java.awt.EventQueue.invokeLater(...)
```

which is the normal Swing pattern for creating the UI on the Event Dispatch Thread.

All graph calculations happen synchronously inside button handlers. For these tiny matrices, the work is effectively instantaneous, so blocking the EDT is not a practical issue.

If levels became large enough for pathfinding to be expensive, the algorithm should move away from the event handler or use a background worker.

## 16. NetBeans GUI Builder

The `.form` files are NetBeans GUI Builder metadata.

They are source files for this particular project, not generated build output, so they remain tracked.

The Java classes contain large `initComponents()` sections generated by the GUI builder. Those sections explain why apparently small screens have fairly large source files.

During cleanup I intentionally did **not** reformat or hand-edit generated layout code just to make the repository look newer. Doing so would add noise without improving the algorithmic story.

## 17. Artwork as part of the game model

The three weighted graphs are stored as image resources:

```text
A.png
A (1).png
A (4).png
```

The map labels and edge weights the player sees are therefore part of static artwork, while the same structure is re-encoded in Java.

That creates another manual synchronization point:

```text
map image
button graph rules
adjacency matrix
```

all have to agree.

A modern rebuild could render nodes and weighted edges directly from `LevelDefinition`, eliminating the static map images as a source of graph truth.

## 18. Repository cleanup

The original commit included normal local NetBeans artifacts:

```text
build/
dist/
nbproject/private/
```

They are useful on one developer machine but not useful as source history.

The cleaned repository removes them and adds ignore rules so future builds do not add them again.

Files intentionally retained include:

```text
build.xml
manifest.mf
nbproject/project.properties
nbproject/project.xml
nbproject/build-impl.xml
lib/CopyLibs/...
```

because those are part of the NetBeans/Ant project definition rather than transient output.

## 19. Build boundary

The project targets Java 8:

```text
javac.source=1.8
javac.target=1.8
```

The permanent GitHub Actions check builds the application through the same Ant project boundary:

```bash
ant clean jar
```

That is preferable to inventing a Maven or Gradle migration solely for portfolio appearance. The repository should still represent what the project actually is.

## 20. No external runtime service

The game is completely local.

It does not need:

- a database;
- an account;
- a network connection;
- an API key;
- a backend;
- a cloud deployment.

The full runtime state lives in the Swing process.

That makes the project a particularly clean example of an algorithm being turned into user interaction without infrastructure distracting from the core idea.

## 21. What would be tested in a rebuild

The current source predates the way I would structure tests today.

A modern implementation would first pull Dijkstra and route state out of Swing, then test them independently.

Minimum algorithm cases:

```text
Map 1 A -> D == 10
Map 2 A -> E == 13
Map 3 A -> F == 15
```

Additional cases:

```text
start == target
unreachable target
single edge
multiple equal shortest paths
zero vertices
invalid start / target indexes
large edge weights
```

Route-session tests would verify:

```text
legal moves only
edge weights come from graph
reset clears route
launch compares to target distance
retry preserves level number
```

Swing UI tests would be the final layer, not the place where algorithm correctness is primarily tested.

## 22. A cleaner architecture

A current rewrite could be small without being over-engineered.

```text
LevelDefinition
      │
      ├────────────► GameScreen
      │                │
      │                ▼
      │            RouteSession
      │
      ▼
ShortestPathSolver
      │
      ▼
ShortestPathResult
├── distance
└── nodes
```

Responsibilities:

### `LevelDefinition`
Owns the graph, labels, source and target.

### `ShortestPathSolver`
Pure Java. Knows nothing about Swing.

### `RouteSession`
Tracks selected nodes and current cost. Reads edge weights from the graph instead of duplicating them.

### `GameScreen`
Renders any level and converts clicks into route actions.

### `ShortestPathResult`
Returns both the optimal distance and the reconstructed node path.

This would keep the original project idea while turning the implementation into something easier to extend and test.

## 23. Why keep the old shape visible?

Because this repository is more useful as a real snapshot of learning than as a fake modern rewrite.

The original version shows the moment where graph theory stopped being just an algorithm exercise and became a small interactive product:

```text
adjacency matrix
      +
Dijkstra
      +
Swing state
      +
visual mission feedback
      =
playable shortest-path exercise
```

The cleanup removes generated clutter, fixes the retry-flow bug, verifies the project still builds, and explains the engineering trade-offs—but it deliberately does not erase the project's history.
