+++
date = '2025-11-19T08:07:44Z'
draft = true
title = 'Exploring Depth First Search'
tags = ['programming', '.net', 'artificial intelligence']
+++

In [my last blog post](../solving-number-problems-with-ai) we developed an MCP server which could solve the Number Round segments from the popular UK gameshow [Countdown](https://en.wikipedia.org/wiki/Countdown_(game_show)). The implementation used an uninformed search algorithm called Breadth First Search. During that implementation I mentioned that while breadth first search will find the optimal solution, it does so at a potentially significant cost. In search spaces where there are many possible next states from an initial state the breadth first search algorithm will fully expand each level of the search space. Using a number round example of 1, 4, 4, 5, 6, 50 and a target of 350 let's explore how big a problem this could be.

In the implementation we explored last time out, we know that the generation of possible states will emit the following options:

+, -, ×, and ÷

| i | j | ithOperandValue | jthOperandValue | Operation | Next State |
|-|-|-|-|-|-|
| 0 | 1 | 1 | 4 | + | 4, 5, 5, 6, 50 |
| 0 | 1 | 1 | 4 | × | 4, 4, 5, 6, 50 |
| 0 | 1 | 1 | 4 | - | 3, 4, 5, 6, 50 |
| 0 | 1 | 1 | 4 | ÷ | 4, 4, 5, 6, 50 |
| 0 | 2 | 1 | 4 | + | 4, 5, 5, 6, 50 |
| 0 | 2 | 1 | 4 | × | 4, 4, 5, 6, 50 |
| 0 | 2 | 1 | 4 | - | 3, 4, 5, 6, 50 |
| 0 | 2 | 1 | 4 | ÷ | 4, 4, 5, 6, 50 |
| 0 | 3 | 1 | 5 | + | 4, 4, 6, 6, 50 |
| 0 | 3 | 1 | 5 | × | 4, 4, 5, 6, 50 |
| 0 | 3 | 1 | 5 | - | 4, 4, 4, 6, 50 |

Note: the exploration of the search space is quite extensive containing over 3,200 rows.

All of the first level of the search space is added to the frontier before any of the nodes on that level are explored. For this problem, there are 50 nodes after the expansion of the first level alone. Each of the next level nodes are placed in the Queue used to store the frontier, during the solve for this problem this queue grows to 1,141 items. The benefit for Breadth First Search is that we find the optimal solution, but it comes at a cost, our memory consumption is typically higher.

## Depth First Search

As we learned last time out, the reason that the Breadth First Search processes the search space in discovery order is that it stores the frontier in a FIFO data structure (specifically a Queue), to cause the Depth First Search behaviour, we need to replace the FIFO data structure with a LIFO data structure i.e. a Stack. That's it, that's all the change we need. By processing the item most recently added to the frontier we are forcing the search space to be explored towards leaf nodes before working back up the levels. We still want to support both strategies so we can compare them, so let's start by adding a simple interface which both of our search strategies will implement.

```csharp
public interface ISearch<TState, TStateTraversalDescription>
{
    IEnumerable<StateTraversal<TState, TStateTraversalDescription>> Execute(TState initialState);
}

public class BreadthFirstSearch<TState, TStateTraversalDescription> : ISearch<TState, TStateTraversalDescription>
    where TStateTraversalDescription : class
{
    // Redacted
}
```

Now, let's add our implementation of the Depth First Search algorithm, it's the same as the Breadth First Search algorithm but uses a Stack instead of a Queue.

```csharp
public class DepthFirstSearch<TState, TStateTraversalDescription> : ISearch<TState, TStateTraversalDescription>
    where TStateTraversalDescription : class
{
    private readonly Predicate<TState> _successIndicator;

    private readonly Func<StateTraversal<TState, TStateTraversalDescription>, IEnumerable<StateTraversal<TState, TStateTraversalDescription>>> _nextStateGenerator;

    private readonly Stack<StateTraversal<TState, TStateTraversalDescription>> _frontier;
    private readonly HashSet<TState> _explored;
    private readonly IEqualityComparer<TState> _stateComparer;

    public DepthFirstSearch(
        Predicate<TState> successIndicator,
        Func<StateTraversal<TState, TStateTraversalDescription>, IEnumerable<StateTraversal<TState, TStateTraversalDescription>>> nextStateGenerator,
        IEqualityComparer<TState> stateComparer)
    {
        _successIndicator = successIndicator;
        _nextStateGenerator = nextStateGenerator;
        _frontier = new Stack<StateTraversal<TState, TStateTraversalDescription>>();
        _explored = new HashSet<TState>(stateComparer);
        _stateComparer = stateComparer;
    }

    public IEnumerable<StateTraversal<TState, TStateTraversalDescription>> Execute(TState initialState)
    {
        _frontier.Push(new StateTraversal<TState, TStateTraversalDescription>(
            null,
            null,
            initialState));

        while (_frontier.TryPop(out var candidate))
        {
            _explored.Add(candidate.Child);

            if (_successIndicator(candidate.Child))
            {
                yield return candidate;
            }

            foreach (var additionalCandidate in _nextStateGenerator(candidate))
            {
                if (!_frontier.Select(f => f.Child).Any(s => _stateComparer.Equals(s, additionalCandidate.Child))
                    && !_explored.Contains(additionalCandidate.Child))
                {
                    _frontier.Push(additionalCandidate);
                }
            }
        }
    }
}
```

As there aren't too many changes, let's briefly discuss them:

- The `_frontier` field is now an instance of `Stack<StateTraversal<TState, TStateTraversalDescription>>`
  - We use `.Push` to add new state traversals onto the frontier
  - We use `.TryPop` when attempting to extract the next state traversal to process from the frontier

That's it, that's all we need to change to move from Breadth First to Depth First. Let's explore this with the help of some diagrams.

69 for the DFS
