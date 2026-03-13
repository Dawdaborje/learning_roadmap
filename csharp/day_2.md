# Day 2 — Powerful C# Features

These are the features that separate C# from "just another OOP language." By the end of today you'll understand why C# developers write so little boilerplate.

--------------------------------------------------------------------------------------------------------------------------------------------------------------

## Generics
You already know generics from Rust. C# works the same way — write code once, use it for any type.

```csharp
// Generic method
public T Max<T>(T a, T b) where T : IComparable<T> {
    return a.CompareTo(b) > 0 ? a : b;
}

Max(3, 7);        // returns 7
Max("apple", "banana");  // returns "banana"

// Generic class
public class Repository<T> where T : class {
    private List<T> _items = new();

    public void Add(T item) => _items.Add(item);
    public T? Find(Func<T, bool> predicate) => _items.FirstOrDefault(predicate);
    public IEnumerable<T> GetAll() => _items;
}

// Use it with any type
var players = new Repository<Player>();
players.Add(new Player("Borje"));
var found = players.Find(p => p.Name == "Borje");
```
The `where` keyword adds constraints — like trait bounds in Rust. Common ones:
```csharp
where T : class          // must be a reference type
where T : struct         // must be a value type
where T : new()          // must have a parameterless constructor
where T : IComparable<T> // must implement an interface
where T : Player         // must be or inherit from Player
```

## LINQ
LINQ (Language Integrated Query) is C#'s killer feature. It lets you query and transform any collection with a chainable, readable API — think Rust iterators but built into the language syntax.

```csharp
var players = new List<Player> {
    new Player("Alice") { Level = 5, Score = 1200 },
    new Player("Bob")   { Level = 3, Score = 800  },
    new Player("Carol") { Level = 5, Score = 950  },
    new Player("Dave")  { Level = 1, Score = 200  },
};

// Filter → sort → transform — reads like English
var topPlayers = players
    .Where(p => p.Level >= 5)           // filter
    .OrderByDescending(p => p.Score)    // sort
    .Select(p => p.Name)                // transform (like .map() in Rust)
    .ToList();
// ["Alice", "Carol"]

// Aggregates
int totalScore = players.Sum(p => p.Score);         // 3150
double avgScore = players.Average(p => p.Score);    // 787.5
Player best = players.MaxBy(p => p.Score);          // Alice
bool anyHighLevel = players.Any(p => p.Level > 4);  // true
bool allActive = players.All(p => p.Score > 0);     // true

// Grouping — incredibly useful
var byLevel = players
    .GroupBy(p => p.Level)
    .Select(g => new { Level = g.Key, Count = g.Count(), Names = g.Select(p => p.Name) });

// foreach (var group in byLevel)
//   Level 5 → Alice, Carol
//   Level 3 → Bob
//   Level 1 → Dave

// First / single / default
var first = players.FirstOrDefault(p => p.Name == "Bob");  // Bob or null
var only = players.SingleOrDefault(p => p.Level == 1);     // Dave (throws if >1 match)
```
LINQ also has a SQL-like query syntax — less common but sometimes clearer:
```csharp
var result = from p in players
             where p.Level >= 3
             orderby p.Score descending
             select p.Name;
```

## Async / Await

C# async/await is the gold standard that Rust and Python both borrowed from. It's built on `Task<T>` (similar concept to Rust's `Future<T>`), but the runtime does all the heavy lifting.

```csharp
// Any method that does I/O should be async
// Return Task for void, Task<T> for a value
public async Task<string> FetchPlayerDataAsync(string playerId) {
    using var client = new HttpClient();

    // await suspends this method, frees the thread, resumes when done
    var response = await client.GetAsync($"https://api.game.com/players/{playerId}");
    response.EnsureSuccessStatusCode();

    return await response.Content.ReadAsStringAsync();
}

// Calling it
var data = await FetchPlayerDataAsync("player_123");

// Run multiple things in parallel
var task1 = FetchPlayerDataAsync("p1");
var task2 = FetchPlayerDataAsync("p2");
var task3 = FetchPlayerDataAsync("p3");

var results = await Task.WhenAll(task1, task2, task3);  // all 3 run concurrently
```
A few important rules:
```csharp
// async propagates up — if you await, your method must be async too
public async Task ProcessAsync() {
    await SomeAsyncOperation();
}

// ConfigureAwait(false) — use in library code to avoid deadlocks
var result = await GetDataAsync().ConfigureAwait(false);

// CancellationToken — always accept one for long-running ops
public async Task LongTaskAsync(CancellationToken ct = default) {
    await Task.Delay(5000, ct);  // cancelled early if token fires
}

// ValueTask — lighter weight when often completing synchronously
public ValueTask<int> GetCachedScoreAsync(string id) {
    if (_cache.TryGetValue(id, out int score))
        return ValueTask.FromResult(score);  // no allocation
    return new ValueTask<int>(FetchScoreAsync(id));
}
```
The big difference from Rust: C# has a runtime (`Task` scheduler built into .NET) so you don't need to choose an executor — it just works.

## Exception Handling
C# uses exceptions rather than `Result<T, E>`. Modern C# style is to use exceptions for truly exceptional conditions, not control flow.
```csharp
// Basic try/catch/finally
try {
    var data = await FetchDataAsync();
    ProcessData(data);
}
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound) {
    // 'when' clause filters which exceptions this catch handles
    Console.WriteLine("Resource not found");
}
catch (HttpRequestException ex) {
    Console.WriteLine($"Network error: {ex.Message}");
}
catch (Exception ex) {
    // Catch-all — be specific when possible
    logger.LogError(ex, "Unexpected failure");
    throw;  // re-throw without losing the stack trace
}
finally {
    // Always runs — for cleanup
    connection.Close();
}

// Custom exceptions — just subclass Exception
public class PlayerNotFoundException : Exception {
    public string PlayerId { get; }

    public PlayerNotFoundException(string playerId)
        : base($"Player '{playerId}' not found") {
        PlayerId = playerId;
    }
}

// Throw it
throw new PlayerNotFoundException("abc123");
```

If you prefer Result-style error handling (coming from Rust), the `FluentResults` or `ErrorOr` NuGet packages are popular for that.

## Delegates & Events
This is C#'s way of passing functions around — equivalent to closures/function pointers in Rust, but with a built-in observer pattern (events) on top.

```csharp
// Func<> — a delegate that returns a value
// Func<TInput, TOutput>
Func<int, int> square = x => x * x;
Func<string, string, string> combine = (a, b) => $"{a} {b}";

// Action<> — a delegate that returns void
Action<string> log = msg => Console.WriteLine(msg);
Action<Player, int> dealDamage = (player, dmg) => player.TakeDamage(dmg);

// Predicate<T> — shorthand for Func<T, bool>
Predicate<Player> isAlive = p => p.Health > 0;

// Pass them around like values
List<int> numbers = new() { 1, 2, 3, 4, 5 };
var evens = numbers.Where(n => n % 2 == 0);  // lambda IS a Func<int, bool>
```

Events build on delegates — the classic observer/pub-sub pattern:

```csharp
public class GameSession {
    // Declare an event — others can subscribe to it
    public event EventHandler<Player>? PlayerJoined;
    public event Action<string>? ChatMessageReceived;

    public void AddPlayer(Player player) {
        // ... add logic ...

        // Raise the event — notify all subscribers
        PlayerJoined?.Invoke(this, player);  // ?. is null-safe: only fires if there are subscribers
    }
}

// Subscribe to events
var session = new GameSession();

session.PlayerJoined += (sender, player) => {
    Console.WriteLine($"{player.Name} joined the session!");
};

session.ChatMessageReceived += msg => Console.WriteLine($"[Chat] {msg}");

// Modern alternative — use Action/Func properties instead of events for simpler cases
public Action<Player>? OnPlayerJoined { get; set; }
```

Let's look at how these all connect in a realistic example:
  ::view-transition-group(*),
  ::view-transition-old(*),
  ::view-transition-new(*) {
    animation-duration: 0.25s;
    animation-timing-function: cubic-bezier(0.19, 1, 0.22, 1);
  }
VvisualizeVvisualize show_widget


Here's a complete example that uses everything from today together:

```csharp
// A real service combining generics, LINQ, async, exceptions, and events
public class PlayerService {
    private readonly Repository<Player> _repo = new();
    
    // Event using a delegate
    public event Action<Player, int>? OnScoreUpdated;

    // Async method with exception handling
    public async Task<Player> GetOrFetchAsync(string id, CancellationToken ct = default) {
        var cached = _repo.Find(p => p.Id == id);
        if (cached != null) return cached;

        try {
            using var client = new HttpClient();
            var json = await client.GetStringAsync($"/api/players/{id}", ct);
            var player = JsonSerializer.Deserialize<Player>(json)
                ?? throw new PlayerNotFoundException(id);
            
            _repo.Add(player);
            return player;
        }
        catch (HttpRequestException ex) {
            throw new PlayerNotFoundException(id) { InnerException = ex };
        }
    }

    // LINQ to get leaderboard
    public IEnumerable<Player> GetLeaderboard(int level, int top = 10) =>
        _repo.GetAll()
             .Where(p => p.Level == level)
             .OrderByDescending(p => p.Score)
             .Take(top);

    // Func<> as a parameter for flexibility
    public void UpdateScore(Player player, int delta, Func<int, int, int>? custom = null) {
        int newScore = custom != null
            ? custom(player.Score, delta)
            : player.Score + delta;
        
        player.Score = newScore;
        OnScoreUpdated?.Invoke(player, newScore);  // fire the event
    }
}

// Wiring it all up
var service = new PlayerService();
service.OnScoreUpdated += (player, score) =>
    Console.WriteLine($"{player.Name} now has {score} points");

var player = await service.GetOrFetchAsync("player_42");
service.UpdateScore(player, 500);
var top5 = service.GetLeaderboard(level: 5, top: 5);
```
