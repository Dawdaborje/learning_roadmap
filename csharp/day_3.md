# Day 3 — Practical Application
Today is about writing C# you'd actually ship. Less theory, more "here's how real C# projects are structured."

## File I/O & JSON

```csharp
// Reading and writing files
using System.IO;
using System.Text.Json;

// Write text
await File.WriteAllTextAsync("log.txt", "Game started\n");

// Append
await File.AppendAllTextAsync("log.txt", "Player joined\n");

// Read all
string content = await File.ReadAllTextAsync("log.txt");

// Read line by line (memory efficient for large files)
await foreach (var line in File.ReadLinesAsync("log.txt")) {
    Console.WriteLine(line);
}

// Work with paths safely — don't concat strings manually
string dir = Path.Combine("data", "players");
string file = Path.Combine(dir, "save.json");
Directory.CreateDirectory(dir);  // creates all missing folders
```
JSON is first-class in modern C# via `System.Text.Json` (built-in, no NuGet needed):
```csharp
public record PlayerSave(string Name, int Level, int Score, DateTime LastSeen);

// Serialize to JSON
var save = new PlayerSave("Borje", 12, 8500, DateTime.UtcNow);
string json = JsonSerializer.Serialize(save, new JsonSerializerOptions {
    WriteIndented = true,          // pretty print
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase  // PascalCase → camelCase
});
await File.WriteAllTextAsync(file, json);

// Deserialize from JSON
string raw = await File.ReadAllTextAsync(file);
var loaded = JsonSerializer.Deserialize<PlayerSave>(raw);

// Deserialize from a stream (better for large files — no full string in memory)
await using var stream = File.OpenRead(file);
var loaded2 = await JsonSerializer.DeserializeAsync<PlayerSave>(stream);
```

Note the `record` keyword above — records are immutable data classes that C# generates equality, `ToString`, and deconstruction for automatically. Use them for data transfer objects and config models.

## HTTP Clients
Never use new `HttpClient()` directly in production — it leaks sockets. Use `IHttpClientFactory` or a static shared instance:

```csharp
// The right way for console apps / small projects
public class GameApiClient {
    // Static shared instance — safe to reuse
    private static readonly HttpClient _http = new() {
        BaseAddress = new Uri("https://api.mygame.com"),
        Timeout = TimeSpan.FromSeconds(10)
    };

    public async Task<Player?> GetPlayerAsync(string id, CancellationToken ct = default) {
        try {
            // GetFromJsonAsync = GET + deserialize in one call (System.Net.Http.Json)
            return await _http.GetFromJsonAsync<Player>($"/players/{id}", ct);
        }
        catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound) {
            return null;
        }
    }

    public async Task<bool> UpdateScoreAsync(string id, int score, CancellationToken ct = default) {
        var payload = new { Score = score };
        var response = await _http.PutAsJsonAsync($"/players/{id}/score", payload, ct);
        return response.IsSuccessStatusCode;
    }
}
```
For ASP.NET or Worker Services, use `IHttpClientFactory` (registered via DI) — which brings us to the next topic.

## Dependency Injection
DI is the backbone of every .NET application. Instead of `new`-ing dependencies everywhere, you register them once and the runtime injects them wherever needed. This is what keeps large codebases testable and modular.
```csharp
// 1. Define your abstractions (interfaces)
public interface IPlayerRepository {
    Task<Player?> GetAsync(string id);
    Task SaveAsync(Player player);
}

public interface ILeaderboardService {
    Task<IEnumerable<Player>> GetTopAsync(int count);
}

// 2. Implement them
public class FilePlayerRepository : IPlayerRepository {
    private readonly string _basePath;

    public FilePlayerRepository(string basePath) {
        _basePath = basePath;
    }

    public async Task<Player?> GetAsync(string id) {
        var path = Path.Combine(_basePath, $"{id}.json");
        if (!File.Exists(path)) return null;
        await using var stream = File.OpenRead(path);
        return await JsonSerializer.DeserializeAsync<Player>(stream);
    }

    public async Task SaveAsync(Player player) {
        var path = Path.Combine(_basePath, $"{player.Id}.json");
        await using var stream = File.Create(path);
        await JsonSerializer.SerializeAsync(stream, player);
    }
}

public class LeaderboardService : ILeaderboardService {
    private readonly IPlayerRepository _repo;  // injected — no new() here

    public LeaderboardService(IPlayerRepository repo) {
        _repo = repo;
    }

    public async Task<IEnumerable<Player>> GetTopAsync(int count) {
        // ... fetch and sort
    }
}
```
```csharp
// 3. Wire it all up in Program.cs
using Microsoft.Extensions.DependencyInjection;

var services = new ServiceCollection()
    // Singleton — one instance for the whole app lifetime
    .AddSingleton<IPlayerRepository>(new FilePlayerRepository("data/players"))
    // Transient — new instance every time it's requested
    .AddTransient<ILeaderboardService, LeaderboardService>()
    // Scoped — one instance per request (mainly for web apps)
    .AddScoped<GameSession>()
    .BuildServiceProvider();

// Resolve and use
var leaderboard = services.GetRequiredService<ILeaderboardService>();
var top10 = await leaderboard.GetTopAsync(10);
```
The three lifetimes to remember: singleton (shared always), transient (fresh every time), scoped (fresh per web request).

## Pattern Matching
Pattern matching is one of C#'s best modern features. It lets you branch on shape, type, and value simultaneously — like a turbo-charged switch.

```csharp
// Type patterns — replaces is + cast
object thing = GetSomething();

string description = thing switch {
    Player p when p.Health == 0  => $"{p.Name} is dead",
    Player p                     => $"{p.Name} has {p.Health}hp",
    Boss b                       => $"Boss {b.Name} phase {b.Phase}",
    string s                     => $"A string: {s}",
    null                         => "Nothing",
    _                            => "Unknown"
};

// Property patterns — match on object shape
string tier = player switch {
    { Level: >= 50, Score: >= 10000 } => "legendary",
    { Level: >= 20 }                  => "veteran",
    { Level: >= 5 }                   => "regular",
    _                                 => "newbie"
};

// Positional patterns (works with records and tuples)
var result = (player.Health, player.Mana) switch {
    (0, _)    => "dead",
    (_, 0)    => "out of mana",
    (<= 20, _) => "critical",
    _          => "healthy"
};

// List patterns (C# 11+)
int[] rolls = { 6, 6, 6 };
bool jackpot = rolls is [6, 6, 6];

string outcome = rolls switch {
    [6, 6, 6]  => "triple six!",
    [6, ..]    => "started strong",
    [.., 1]    => "ended poorly",
    _          => "normal roll"
};
```

## Mini Project — CLI Leaderboard Tool
Let's put everything together. A small CLI tool that manages a player leaderboard stored as JSON.

```csharp
// Program.cs — the full app, ~80 lines

using System.Text.Json;
using Microsoft.Extensions.DependencyInjection;

// ── Models ────────────────────────────────────────────────────────
record Player(string Id, string Name, int Score, int Level) {
    public Player WithScore(int newScore) => this with { Score = newScore };
}

// ── Repository ────────────────────────────────────────────────────
interface IPlayerRepo {
    Task<List<Player>> LoadAllAsync();
    Task SaveAllAsync(List<Player> players);
}

class JsonPlayerRepo : IPlayerRepo {
    const string Path = "leaderboard.json";
    static readonly JsonSerializerOptions Opts = new() { WriteIndented = true };

    public async Task<List<Player>> LoadAllAsync() {
        if (!File.Exists(Path)) return new();
        await using var s = File.OpenRead(Path);
        return await JsonSerializer.DeserializeAsync<List<Player>>(s) ?? new();
    }

    public async Task SaveAllAsync(List<Player> players) {
        await using var s = File.Create(Path);
        await JsonSerializer.SerializeAsync(s, players, Opts);
    }
}

// ── Service ───────────────────────────────────────────────────────
class LeaderboardService {
    readonly IPlayerRepo _repo;
    public event Action<Player>? OnScoreUpdated;

    public LeaderboardService(IPlayerRepo repo) => _repo = repo;

    public async Task AddPlayerAsync(string name) {
        var players = await _repo.LoadAllAsync();
        if (players.Any(p => p.Name == name)) {
            Console.WriteLine($"Player '{name}' already exists."); return;
        }
        var id = Guid.NewGuid().ToString()[..8];
        players.Add(new Player(id, name, 0, 1));
        await _repo.SaveAllAsync(players);
        Console.WriteLine($"Added player '{name}' (id: {id})");
    }

    public async Task AddScoreAsync(string name, int points) {
        var players = await _repo.LoadAllAsync();
        var idx = players.FindIndex(p => p.Name == name);
        if (idx < 0) { Console.WriteLine($"Player '{name}' not found."); return; }

        var updated = players[idx].WithScore(players[idx].Score + points);
        players[idx] = updated;
        await _repo.SaveAllAsync(players);
        OnScoreUpdated?.Invoke(updated);
    }

    public async Task ShowLeaderboardAsync(int top = 10) {
        var players = await _repo.LoadAllAsync();
        var ranked = players
            .OrderByDescending(p => p.Score)
            .Take(top)
            .Select((p, i) => (Rank: i + 1, Player: p));

        Console.WriteLine("\n── Leaderboard ─────────────────");
        foreach (var (rank, p) in ranked) {
            string medal = rank switch { 1 => "🥇", 2 => "🥈", 3 => "🥉", _ => "  " };
            Console.WriteLine($"{medal} #{rank,-3} {p.Name,-20} {p.Score,8} pts  Lv{p.Level}");
        }
        Console.WriteLine("────────────────────────────────\n");
    }
}

// ── Entry point ───────────────────────────────────────────────────
var services = new ServiceCollection()
    .AddSingleton<IPlayerRepo, JsonPlayerRepo>()
    .AddSingleton<LeaderboardService>()
    .BuildServiceProvider();

var lb = services.GetRequiredService<LeaderboardService>();
lb.OnScoreUpdated += p => Console.WriteLine($"  ↑ {p.Name} now has {p.Score} pts");

// Parse args and dispatch
string command = args.ElementAtOrDefault(0) ?? "help";

await (command switch {
    "add"   => lb.AddPlayerAsync(args.ElementAtOrDefault(1) ?? "Unknown"),
    "score" => lb.AddScoreAsync(
                   args.ElementAtOrDefault(1) ?? "",
                   int.TryParse(args.ElementAtOrDefault(2), out var n) ? n : 0),
    "top"   => lb.ShowLeaderboardAsync(),
    _       => Task.Run(() => Console.WriteLine(
                   "Usage:\n  dotnet run add <name>\n  dotnet run score <name> <pts>\n  dotnet run top"))
});
```

Run it:
```bash
dotnet run add Borje
dotnet run add Alice
dotnet run score Borje 1500
dotnet run score Alice 800
dotnet run score Borje 300
dotnet run top

── Leaderboard ─────────────────
🥇 #1   Borje                1800 pts  Lv1
🥈 #2   Alice                 800 pts  Lv1
────────────────────────────────
```

Every concept from all 3 days is in there — records, LINQ, async/await, DI, events, pattern matching, file I/O, and JSON.
Where to Go From Here
  ::view-transition-group(*),
  ::view-transition-old(*),
  ::view-transition-new(*) {
    animation-duration: 0.25s;
    animation-timing-function: cubic-bezier(0.19, 1, 0.22, 1);
  }
VvisualizeVvisualize show_widget

Click any path in the diagram to go deeper. A few honest takes given your background:
vs Rust — C# is dramatically faster to write. No borrow checker, no lifetime annotations, garbage collection handles memory. You lose control over allocations and performance ceilings, but for anything that isn't a game engine or systems software, that tradeoff is completely worth it. Your Django instincts map cleanly to ASP.NET Core.
vs Python — C# is stricter but the tooling pays you back. Refactoring in Rider or VS Code with the C# extension is a completely different class of experience from Python — rename a class, every file updates correctly. The type system catches entire categories of bugs before you run anything.
Tooling worth installing now — JetBrains Rider (best-in-class IDE, free for non-commercial) or VS Code with the C# Dev Kit extension. Either gives you inline errors, auto-imports, and a debugger that actually works.
