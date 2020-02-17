# A Go Roguelike

I'm building a [roguelike](https://en.wikipedia.org/wiki/Roguelike) game in Go called ["Jumpdrive."](https://imgur.com/kYE6CvR)

It's a 70's space sci-fi about a pilot who's crashed her scout craft onto an island on an alien planet.
Why?
For me, drinking a bourbon while building a silly videogame in a pleasant language evokes the same sort of relaxed-but-rewarding state of mind that playing videogames does.

![screenshot](https://i.imgur.com/9Z7PpsO.png)

These are a few thoughts I've had about programming while hacking it together.

## Generic Grids

Roguelikes are all about grids. They're rendered into ASCII or ASCII-like grids of tiles.
Movement and action is all oriented to the north, south, east, or west.
The procedural generation of levels - cellular automata, fractal terrain, etc - uses grid-based algorithms.

Ideally, we could define a generic grid:

```go
type Grid (type E) struct {
	cells []E
	width int
	height int
}

func New (type E) (w, h int) Grid(E) {
	return Grid(E){
		width: w,
		height: h,
		cells: make([]E, w * h),
	}
}
```

Which could be used in a variety of grid types:

```go
type Tile struct {
	Terrain int
	Prop int
	Light color.RGBA
}

tileGrid := grid.New(Tile)(128, 128)
```

And each of those grid types would support all the handy features of `Grid`, like:

```go
neighbors := tileGrid.Neighbors4(16, 24)    // []Tile

tile, wrapped := tileGrid.At(192, 100)      // Tile, bool

continent := tileGrid.Contiguous(10, 20, func(t Tile) bool {    // []Tile
	return t.Terrain == terrain.Rock
})
```

### Reality

Unfortunately, that isn't possible in Go ([yet!](https://blog.golang.org/why-generics)).
I spun my wheels for a while finding a workaround that I was satisfied with before
landing on a composable grid indexing system.

First, you pair a generic slice with a grid:

```go
type TileGrid struct {
	grid.Grid
	V []Tile
}

func NewTileGrid(w, h int) TileGrid {
	return TileGrid{
		Grid: grid.Grid{Width: w, Height: h},
		V:    make([]Tile, w * h),
	}
}

tileGrid := NewTileGrid(128, 128)
```

Then, all of the grid's methods return indices for the attached slice:

```go
nn := tileGrid.Neighbors4(16, 24)    // []int

i, wrapped := tileGrid.Index(192, 100)   // int, bool

cc := tileGrid.Contiguous(i, func(i int) bool {  // []int
	return tileGrid.V[i].Terrain == terrain.Rock
})
```

For many scenarios, just the index, or even the length, is sufficient.
If not, this solution grows more awkward:

```go
nn := tileGrid.Neighbors4(16, 24)    // []int

neighbors := make([]Tile, len(nn))
for i, j := range nn {
	neighbors[i] = tileGrid.V[j]
}
```

While I'm proud of what I believe is a simple and pragmatic workaround,
I look forward to ripping this code out and replacing it with generic containers.
If you can recommend a better approach, please do - I'm not a Go expert and I would be happy to improve it.

In Jumpdrive, this grid system is used in the procedural generation pipeline that
starts with a fractal noise, discretizes it into types of contiguous terrain on an island,
tunnels rooms and mazes through it, builds structures and props, and finally adds items and creatures:

![fractal noise](https://i.imgur.com/i2f99w4.png)

![island terrain](https://i.imgur.com/mIQgH0S.png)

![structures](https://i.imgur.com/Uk8wXUA.png)

![final map](https://i.imgur.com/9i29mt4.png)

The grid package is also used in Jumpdrive's path-traced renderer, which is designed to evoke the feeling of ASCII
with extra color, light, and [drama](https://i.imgur.com/kYE6CvR.mp4):

![lighting](https://i.imgur.com/6V13Hx7.png)
![more lighting](https://i.imgur.com/E5BQ72i.png)

## Packages, Seams, and Structure

When learning new codebases, I appreciate when the code itself acts as a sort of guide:
`package main` as the table of contents, wiring up the program's structure at the highest level of abstraction
and describing the system's overall shape. `main` shouldn't drop down into detailed work,
but neither should it obfuscate what the program does.

Here is `cmd/jumpdrive/jumpdrive.go`:

```go
// imports omitted

const (
	title      = "Jumpdrive"
	pixelScale = 3.0
	worldSize  = 513
)

var (
	winSize    = image.Point{600, 400}
	fullscreen = flag.Bool("full", false, "run in fullscreen")
	seed       = flag.Int64("seed", 1, "random seed (1)")
	profCPU    = flag.Bool("cpu", false, "record a CPU profile")
)

func main() {
	flag.Parse()

	if err := play(); err != nil {
		log.Fatalf("error: %v", err)
	}
}

func play() error {
	rng := rand.New(rand.NewSource(*seed))

	gm, err := game.New(rng, worldSize)
	if err != nil {
		return fmt.Errorf("creating game: %w", err)
	}

	win, err := newWindow(gm, *fullscreen)
	if err != nil {
		return fmt.Errorf("creating window: %w", err)
	}

	if *profCPU {
		defer profile.Start().Stop()
	}

	return win.OpenAndRun()
}

func newWindow(gm game.Instance, full bool) (win *ui.Window, err error) {
	if full {
		return ui.NewFullWindow(gm, title, pixelScale)
	}
	return ui.NewWindow(gm, title, pixelScale, winSize)
}
```

I've been very happy with how naturally seams have emerged between packages due to Go's `package.Export` organization.
The game's logic knows nothing about the UI that's rendering it,
the terrain generator knows nothing about the level it's populating, and so forth.
Dependencies point from the UI (less stable) towards game logic (more stable) to small underlying systems like the ECS (most stable).

Go forces me to sit and think about names and groupings that will provide a nice API, which in turn
leads me to better choices than I would make otherwise (*"plop the Foo class into utils/Foo.lang and then import ../../utils/Foo"*).

Several packages have become completely decoupled; I plan to split them into their own modules for publishing:

- grid
- ecs (Entity-Component System)
- sprites (with Ebiten)
- diamondsquare (terrain generation)

For the first time, I'm using an `internal` package.
I don't know if I'm using it correctly, but there were several packages that all relied on agreeing
on certain measurements with each other, and ensuring that these constants made it down dependency chains
was a brittle solution:

```go
package internal

const (
	TileWidth    = 16
	TileHeight   = 16
	SpriteWidth  = 16
	SpriteHeight = 24
)
```

## Entity-Component Systems

If you haven't done much game development, [entity-component systems](https://en.wikipedia.org/wiki/Entity_component_system) might be new to you.
On the other hand, if you've ever worked in Unity, then whether you know it or not you've built on top of an ECS.

Games tend to have many more object instances, as well as many more types of objects, than business-oriented applications.
Additionally, the number of other types a given object type should interact with tends to be higher.
For example, a "bullet" should interact with a "monster." But it should also interact with a "player."
As you add more things to a game, you'll find that many of them should also interact with "bullet:"
"wall," "barrel," "vehicle," etc.

The exponential increase of these interactions as you linearly increase the types of things in your game
tends to lead game code into a complexity wall pretty quickly.
That punishes creativity, since every new idea comes with exponentially more pain buried in its implementation.

### A naive solution

To flatten out that complexity into something more linear, game developers turn to orthogonal behaviors via components.
In a language like Go, you might first try to implement this pattern via broad compositional structures (that's what I did, anyway):

```go
type Player struct {
	HealthComponent
	PerceptionComponent
	PositionComponent
	WalkComponent
}

type Bullet struct {
	DamageComponent
	PositionComponent
	FlyComponent
}
```

This actually works okay, but brings several tradeoffs.

In an entity-component system, order is important. All entities that can move, should move, and then
all entities that can be damaged by weapons, should be damaged by weapons touching them.
So you can't just loop through each object and apply all of its components, then do the same for the next object, etc.

Enforcing order with embedded structs over a large set of non-homogenous objects gets awkward:

```go
type Entity interface{}

func Update(entities []Entity) {
	for _, e := range entities {
		if h, ok := e.(*HealthComponent); ok {
			h.UpdateHealth()
		}
	}
	for _, e := range entities {
		if p, ok := e.(*PerceptionComponent); ok {
			p.UpdatePerception()
		}
	}
	// ...etc
}
```

That kind of pattern is slow, and made even slower by encouraging cache misses - the data is stored per-entity but our pattern of access is per-component.

Game state also needs to be saved and loaded; serializing these embedded components is going to be a pain. Similar concerns exist for sharing game state over a network.

Additionally, these statically-typed entities are inflexible at runtime; any component that might ever be available on an entity must be included in its type - so if your player could ever fly, `Player` needs the `FlyComponent`,
plus a mechanism for marking components as disabled. It's a lot of book-keeping and the proliferation of "it could happen"
components hurts readability.

### package ecs

Instead, Jumpdrive now uses a minimal Entity-Component System package that I modeled after the most useful parts of Unity's
behaviors. Each `Entity` in the `System` has a name, an ID, and zero or more tags; beyond that, all behavior is controlled by Components:

```go
sys.Create("player",
	&Player{},
	&Position{Point: pt},
	&Perceptive{Range: 48},
	&Harmable{Health: 3},
	&Inventory{},
).Tag("creature")
```

A tag is essentially a no-op component that simplifies searching for and building lists of entities.

Notice that a "crab" isn't so different from a "player:"

```go
sys.Create("crab",
	&Wander{Towards: gen.Water, Rng: rng},
	&Position{Point: pt},
	&Perceptive{Range: 32},
	&Harmable{Health: 2},
).Tag("creature")
```

A ray gun can interact with both a player and a crab.
A player can pick it up, due to its `Inventory` component interacting with the gun's `Collectable` component.
A crab can be damaged by the gun via the interactions between the `Shoot` component and the `Harmable` component:

```go
sys.Create("ray-gun",
	&Shoot{Ammo: "ray", Damage: 1, Range: 100},
	&Position{Point: pt},
	&Collectable{},
)
```

![shooting](https://i.imgur.com/R4v05S8.png)

Interestingly, without any extra code - or even meaning to do it - players can also shoot themselves since they,
too, include `Harmable`.

Rather than suppress creativity, an ECS tends to encourage and inspire creativity with emergent behaviors.
Combining existing components in new ways creates new interactions without any new code!

What about order? One solution would be to require each `Component` to specify its priority in the queue.
However, I prefer explicitness, so each `System` has its own stack of components.
They bear some similarity to the stacks of middleware that web developers may be more familiar with:

```go
sys := ecs.NewSystem(
	InputComponent,
	WorldComponent,
	WeatherComponent,
	ShootComponent,
	HarmableComponent,
	PlayerComponent,
	WanderComponent,
	PositionComponent,
	InventoryComponent,
	CollectableComponent,
	PerceptiveComponent,
	BlockingComponent,
)
```

New components are easy to add to the system, which then triggers their behaviors with events.
For example, whenever `system.Trigger("Step")` is called, all `Step` methods on all components
are executed, in system-wide component order:

```go
const PlayerComponent = "Player"

type Player struct{}

func (p *Player) Name() ecs.ComponentName {
	return PlayerComponent
}

func (p *Player) Step(s *ecs.System, e *ecs.Entity) {
	input := s.ComponentNamed(InputComponent).(*Input)

	h := e.Component(HarmableComponent).(*Harmable)
	if h.Health <= 0 {
		return
	}

	switch cmd := input.lastCommand.(type) {
	case MoveCommand:
		p.move(s, e, cmd)
	case ShootCommand:
		p.shoot(s, e, cmd)
	}
}
```

The ecs package also provides several high-performance methods to query the state of the system,
which is stored as maps of components rather than slices of entities:

```go
player := sys.EntityNamed("player")

pos := player.Component(game.PositionComponent).(*game.Position)

world := sys.ComponentNamed(WorldComponent).(*World)

items := sys.EntitiesWith(game.CollectableComponent)

creatures := sys.EntitiesTagged("creature")
```

So, that's the hack project that I've been inching closer and closer towards a usable state over the past couple of months. I learned today that [Diablo started as a roguelike with custom lighting](https://www.youtube.com/watch?v=VscdPA6sUkc) 24 years ago, so while this may not be cutting-edge, at least I'm in good company :)

## Considerations
- Make grid an entity, as it is a game object after all

## Next steps

- Some cleanup inspired by [Clean Code](https://www.amazon.com/Clean-Code-Handbook-Software-Craftsmanship/dp/0132350882), especially in the very-messy rendering system.
- Leveraging the path-traced lighting system (animated ray-gun rays, foliage that catches on fire, bioluminescence...)
- Leveraging the ECS (map, teleport, drone items; enemy aliens, combat tactics; saving and restoring state...)
- Leveraging the grid (procgen terrain, prop variety; hiding valuable items in hard-to-reach cells...)

If you have ideas, corrections, or resources, please share them: [@hunterloftis](https://twitter.com/hunterloftis)


