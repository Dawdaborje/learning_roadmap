# Day 1 — Language Fundamentals

## Setup
Install the .NET SDK from dotnet.microsoft.com, then:

```
dotnet new console -n HelloCsharp
cd HelloCsharp
dotnet run
```
That's it — no Cargo.toml drama, no virtualenvs. C# uses `.csproj` files under the hood.

Types & Variables
C# is statically typed like Rust, but garbage collected. No ownership, no lifetimes.

```csharp
// Explicit types
int age = 30;
double price = 9.99;
bool active = true;
string name = "Borje";

// Type inference (like Rust's let)
var count = 42;          // compiler infers int
var message = "hello";   // compiler infers string

// Constants
const double Pi = 3.14159;

// Nullable types (very important in C#)
int? maybeNull = null;   // the ? makes any value type nullable
string? nullableStr = null;
```

Compared to Rust: `var` is like `let`, but mutable by default (no `mut` needed). Nullability is opt-in with `?`.

## Control Flow
```csharp
// If / else — same as Rust
if (age > 18) {
    Console.WriteLine("Adult");
} else {
    Console.WriteLine("Minor");
}

// Switch expressions (modern C# — very clean)
string label = age switch {
    < 13  => "child",
    < 18  => "teen",
    < 65  => "adult",
    _     => "senior"    // _ is the default, like Rust's _
};

// For loops
for (int i = 0; i < 10; i++) { }

// Foreach (most common — use this over for when iterating)
var items = new List<string> { "a", "b", "c" };
foreach (var item in items) {
    Console.WriteLine(item);
}

// While
while (count > 0) { count--; }
```

## Classes & Interfaces
This is where C# shines. It's a full OOP language — think of it as Python's classes but with type safety baked in.

```csharp
// A basic class
public class Player {
    // Properties (not raw fields — C# convention)
    public string Name { get; set; }
    public int Health { get; private set; } = 100;  // settable only internally

    // Constructor
    public Player(string name) {
        Name = name;
    }

    // Method
    public void TakeDamage(int amount) {
        Health = Math.Max(0, Health - amount);
    }

    // Override ToString (like Python's __repr__)
    public override string ToString() => $"{Name} ({Health}hp)";
}

// Interfaces — define a contract, no implementation
public interface IDamageable {
    void TakeDamage(int amount);
    int Health { get; }
}

// Inheritance
public class Boss : Player, IDamageable {
    public int Phase { get; private set; } = 1;

    public Boss(string name) : base(name) { }

    // Override a method
    public override string ToString() => $"[BOSS] {base.ToString()} Phase {Phase}";
}
```


C# uses single inheritance for classes (like Java), but you can implement multiple interfaces. No traits like Rust, but interfaces get you 90% of the way there. Modern C# also supports default interface methods for the remaining 10%.

## Collections

```csharp
// List<T> — the workhorse (like Vec<T> in Rust)
var scores = new List<int> { 10, 20, 30 };
scores.Add(40);
scores.Remove(10);

// Dictionary<K, V> — like Rust's HashMap
var playerStats = new Dictionary<string, int> {
    ["strength"] = 10,
    ["dexterity"] = 8,
};
playerStats["luck"] = 5;

// Access with safety
if (playerStats.TryGetValue("wisdom", out int val)) {
    Console.WriteLine(val);
}

// HashSet<T> — unique values only
var visited = new HashSet<string>();
visited.Add("zone_1");
```

## Null Handling
Null is C#'s most infamous feature — and modern C# (10+) has good tools to tame it.

```csharp
string? maybeName = GetName();  // could be null

// Null conditional operator — short-circuits to null if left side is null
int? length = maybeName?.Length;

// Null coalescing — provide a fallback (like Rust's unwrap_or)
string display = maybeName ?? "Anonymous";

// Null coalescing assignment
maybeName ??= "Default";

// Null forgiving operator — tells compiler "trust me, not null"
// Use sparingly — it's a code smell
string definitelyName = maybeName!;
```

That's Day 1. The key takeaways: C# is verbose but clear, classes are first-class citizens, and null safety is opt-in but well worth enabling.
