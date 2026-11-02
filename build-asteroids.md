# Building Asteroids

> **Source / Reference:** [Boot.dev — Build Asteroids](https://www.boot.dev/courses/build-asteroids)
> These are my personal notes from the Boot.dev course. All credit for the course material goes to Boot.dev.
>
> **Started:** 2026-11-01
> **Progress so far:** FPS / delta time, game loop, sprite groups, player movement, asteroid field, shooting, collision, splitting asteroids

This note is my "oh, that's why it works like that" reference. It explains the whole loop from creating objects to moving and drawing them, using the actual code in the `asteroids` repo.

---

## FPS

As it stands, our game is re-drawing the screen as fast as it possibly can. This causes it to use a lot more CPU than it actually needs!

FPS stands for [frames per second](https://en.wikipedia.org/wiki/Frame_rate). We restrict our game to draw a maximum of 60 times per second, or 60 FPS:

```python
dt = clock.tick(60) / 1000
```

`clock.tick(60)` **throttles** the loop (it sleeps if a frame finished early) and **returns** the milliseconds since the last call. Dividing by 1000 gives seconds.

So 60 is a *ceiling*, not a guarantee:

- Frame finishes in 5 ms → `tick` waits ~11.7 ms → ~60 FPS
- Frame takes 30 ms → no waiting → ~33 FPS, and `dt` is larger

The loop can always run *slower* than 60 FPS, never faster.

---

## Delta Time

The Greek letter delta (Δ) is often used to represent a change in a value in mathematics. In game development, we use "delta time" to represent the amount of time that has passed since the last frame was drawn. This value is useful to decouple the game's speed from the speed it's being drawn to the screen.

If your computer's CPU speeds up (its speed is not constant, like a sprinter running as fast as they can), the asteroids shouldn't also speed up. Conversely, if your computer slows down, the asteroids shouldn't also slow down: they may just move less smoothly.

So instead of:

```python
self.position += self.velocity          # speed depends on FPS
```

we do:

```python
self.position += self.velocity * dt     # speed is per-second, FPS independent
```

At 60 FPS, `dt ≈ 0.0167`. At 30 FPS, `dt ≈ 0.0333`. Either way, an object with velocity `(0, 200)` moves **200 pixels in one real second**.

`dt` is computed **at the end** of the loop, so it describes the frame that just finished and is used by the **next** iteration. On the very first frame `dt` is still `0.0`, so nothing moves yet — that's normal.

---

## The Game Loop

Every game is basically two phases.

### Phase 1 — Setup (once, before the loop)

Create the groups, tell each class which groups it belongs to, then create the objects:

```python
updatable = pygame.sprite.Group()
drawable  = pygame.sprite.Group()
asteroids = pygame.sprite.Group()
shots     = pygame.sprite.Group()

Player.containers = (updatable, drawable)
Asteroid.containers = (asteroids, updatable, drawable)
AsteroidField.containers = updatable
Shot.containers = (shots, updatable, drawable)

player = Player(SCREEN_WIDTH / 2, SCREEN_HEIGHT / 2)
af = AsteroidField()
```

Nothing is drawn or moved yet. These lines only build **lists of object references**:

```
updatable -> [player, asteroidfield, (later: asteroids, shots)]
drawable  -> [player, (later: asteroids, shots)]
asteroids -> [(later: asteroids)]
shots     -> [(later: shots)]
```

The groups remember the objects for us. After this, we never have to call `player.update()` or `player.draw()` by name.

### Phase 2 — The loop (runs up to 60 times per second)

Each pass = **one frame** = one still image in an animation.

```python
while True:
    log_state()                                   # 1. debug log
    for event in pygame.event.get():              # 2. one-shot events (quit)
        if event.type == pygame.QUIT:
            return
    screen.fill("black")                          # 3. erase the canvas
    updatable.update(dt)                          # 4. change world state
    # ...collision checks...                      # 5. react to state
    for obj in drawable:                          # 6. paint world state
        obj.draw(screen)
    pygame.display.flip()                         # 7. show the frame
    dt = clock.tick(60) / 1000                    # 8. measure frame time
```

Two rules that matter:

1. **`update` first, `draw` second.** Compute the new world, then paint it.
2. **`update` changes state; `draw` only reads state.** Movement lives in `update`. `draw` never moves anything.

---

## Grouping Objects (`pygame.sprite.Group`)

This was the part that confused me: how does `updatable.update(dt)` reach the player without referencing `player`?

### How an object joins a group

`Player.containers` is a **class attribute** (set before any player exists). When we do `Player(x, y)`:

```
Player.__init__
  -> CircleShape.__init__
       -> if hasattr(self, "containers"):
              super().__init__(*self.containers)   # Sprite.__init__(updatable, drawable)
```

`pygame.sprite.Sprite.__init__` sees the groups and calls `self.add(updatable, drawable)`, inserting `self` into each group. The group stores a **reference** to the object, not a copy.

### How the group calls it

`pygame.sprite.Group.update` is just a loop:

```python
def update(self, *args, **kwargs):
    for sprite in self.sprites():
        sprite.update(*args, **kwargs)
```

So:

```python
updatable.update(dt)
# becomes:
# player.update(dt)
# asteroidfield.update(dt)
# asteroid.update(dt)   (for each asteroid)
# shot.update(dt)       (for each shot)
```

That's polymorphism: every object has its own `update(dt)` and `draw(screen)`, and the loop treats them all the same. The variable `player` is only *one way* to reach the object; the groups are another way to reach the same object.

```
player (variable) ─┐
                   ├──> same Player object ──> in updatable group
                   └──> same Player object ──> in drawable group
```

### Gotcha

`containers` is a class attribute. Set it **before** creating instances. If you set `Player.containers` after creating a player, that existing player will **not** be added.

Also: `pygame.sprite.Group` is separate from `asteroids` / `shots`. The latter are just extra groups so we can answer "which things are asteroids?" or "which things are shots?" for collision checks.

---

## The Base Class: `CircleShape`

```python
class CircleShape(pygame.sprite.Sprite):
    containers: tuple[pygame.sprite.Group, ...]

    def __init__(self, x, y, radius):
        if hasattr(self, "containers"):
            super().__init__(*self.containers)
        else:
            super().__init__()
        self.position = pygame.Vector2(x, y)
        self.velocity = pygame.Vector2(0, 0)
        self.radius = radius

    def draw(self, screen): pass    # must override
    def update(self, dt): pass      # must override

    def collides_with(self, other):
        distance = pygame.math.Vector2.distance_to(self.position, other.position)
        return distance <= self.radius + other.radius
```

Why it exists:

- **Inheritance** — `Player`, `Asteroid`, `Shot` are all `CircleShape`, so they share `position`, `velocity`, `radius`, group registration, and collision logic.
- **Interface** — every child must provide `draw` and `update` (the base versions are empty stubs).
- **Collision** — `collides_with` is generic: two circles collide when the distance between centers is less than or equal to the sum of their radii.

`Player` and `Asteroid` and `Shot` "are a" `CircleShape`. That's the "A should only inherit from B if A is always a B" rule from the OOP notes.

---

## Moving Objects (all in `update`)

### Player — `player.py`

```python
def update(self, dt):
    keys = pygame.key.get_pressed()
    if keys[pygame.K_a]:
        self.rotate(dt * -1.0)
    if keys[pygame.K_d]:
        self.rotate(dt)
    if keys[pygame.K_w]:
        self.move(dt)
    if keys[pygame.K_s]:
        self.move(-dt)
    if keys[pygame.K_SPACE]:
        self.shoot_timer -= dt
        self.shoot()

def rotate(self, dt):
    self.rotation += PLAYER_TURN_SPEED * dt

def move(self, dt):
    unit_vector = pygame.Vector2(0, 1)
    rotated_vector = unit_vector.rotate(self.rotation)
    rotated_with_speed_vector = rotated_vector * PLAYER_SPEED * dt
    self.position += rotated_with_speed_vector
```

Key idea:

- `self.rotation` is an **angle in degrees**. Turning only changes the angle.
- The player has no "facing vector" stored. Instead, `move` takes a default forward vector `(0, 1)`, rotates it by `self.rotation`, scales by speed and `dt`, and adds it to `position`.
- Because `keys[...]` is checked every frame, holding `W` moves a little each frame → smooth motion.

Held keys are read with `pygame.key.get_pressed()` inside `update`, **not** from `pygame.event.get()` (which only reports one-shot events like key-down/up and quit).

### Asteroid — `asteroid.py`

```python
def update(self, dt):
    self.position += self.velocity * dt
```

No input, no turning — just "keep drifting in the direction and speed we were given."

### Shot — `shot.py`

```python
def update(self, dt):
    self.position += self.velocity * dt
```

Exactly like an asteroid. Bullets are just fast, tiny circles.

### AsteroidField — `asteroidfield.py`

The field is not a `CircleShape`; it's a plain `pygame.sprite.Sprite` that never gets drawn. Its job is spawning:

```python
def update(self, dt):
    self.spawn_timer += dt
    if self.spawn_timer > ASTEROID_SPAWN_RATE_SECONDS:
        self.spawn_timer = 0
        edge = random.choice(self.edges)
        speed = random.randint(40, 100)
        velocity = edge[0] * speed
        velocity = velocity.rotate(random.randint(-30, 30))
        position = edge[1](random.uniform(0, 1))
        kind = random.randint(1, ASTEROID_KINDS)
        self.spawn(ASTEROID_MIN_RADIUS * kind, position, velocity)
```

- `spawn_timer` accumulates `dt`. When it passes the spawn rate, it resets and spawns one asteroid.
- `edges` is a list of `(inward_direction, position_function)` pairs, one per screen side.
  - The direction points **into** the screen, so the asteroid travels inward.
  - The lambda takes a random `0..1` and returns a point just **outside** that edge (by `ASTEROID_MAX_RADIUS`), so asteroids appear off-screen and drift in.
- `velocity.rotate(random.randint(-30, 30))` adds a little random angle so they don't all travel perfectly straight.
- `kind` = 1, 2, or 3 → radius 20, 40, or 60.

`spawn()` creates the `Asteroid` and sets its velocity **after** construction. Creating the object is enough to register it in the `asteroids`, `updatable`, and `drawable` groups (via `Asteroid.containers`).

---

## Drawing Objects (all in `draw`)

Drawing only **reads** `position`, `rotation`, `radius` and paints. It never modifies them.

### Player

```python
def draw(self, screen):
    pygame.draw.polygon(screen, "white", self.triangle(), LINE_WIDTH)

def triangle(self):
    forward = pygame.Vector2(0, 1).rotate(self.rotation)
    right = pygame.Vector2(0, 1).rotate(self.rotation + 90) * self.radius / 1.5
    a = self.position + forward * self.radius
    b = self.position - forward * self.radius - right
    c = self.position - forward * self.radius + right
    return [a, b, c]
```

- `forward` is the direction the player faces, derived from `rotation`.
- `right` is `forward` rotated 90°, scaled down to make the back corners narrower.
- The three points are computed **relative to the current position**, so when `update` moves the player, the next `draw` automatically paints the triangle at the new spot.
- The player's shape is never stored as pixels — it's recomputed from `position` + `rotation` every frame. This is why rotating and moving look consistent.

### Asteroid and Shot

```python
pygame.draw.circle(screen, "white", self.position, self.radius, LINE_WIDTH)
```

Both are hollow white circles; only the radius differs (`20/40/60` vs `5`).

---

## One Frame, Traced

Player at `(640, 360)`, `rotation = 0`, facing down. Holding `W`. `PLAYER_SPEED = 200`, `dt ≈ 0.0167`.

| Step | What happens | Result |
|---|---|---|
| `screen.fill` | blank canvas | — |
| `updatable.update` → `Player.update` | `move(dt)`: forward `(0,1)` → `(0,1) * 200 * 0.0167 = (0, 3.34)` | position `(640, 363.34)` |
| `updatable.update` → `Asteroid.update` | each `position += velocity * dt` | asteroids drift |
| `updatable.update` → `Shot.update` | each `position += velocity * dt` | bullets fly |
| `updatable.update` → `AsteroidField.update` | `spawn_timer += dt` | maybe spawns |
| collision checks | `collides_with` on asteroid vs player / shot | maybe kill / split |
| `drawable.draw` | triangle recomputed at `(640, 363.34)` | painted there |
| `display.flip` | show the frame | looks moved |
| `clock.tick(60)` | returns ~16.7 ms | `dt` for next frame |

After ~60 frames (one second), the player has moved 200 px down. At 144 FPS it takes 144 frames but still moves 200 px per second — that's what `* dt` gives us.

Think of each loop as one sheet of animation paper: erase, move the characters a tiny bit, draw them where they now are, flip to the new sheet, repeat. Motion is an illusion made of many still frames.

---

## Shooting

```python
def shoot(self):
    if self.shoot_timer > 0:
        return
    else:
        self.shoot_timer = PLAYER_SHOOT_COOLDOWN_SECONDS

    shot = Shot(self.position.x, self.position.y)
    shot.velocity = pygame.Vector2(0, 1).rotate(self.rotation) * PLAYER_SHOOT_SPEED
```

- The shot spawns at the player's current center.
- Its velocity is the player's facing vector scaled by `PLAYER_SHOOT_SPEED` (500), so it moves ~500 px per second.
- Without the cooldown, holding space would create a bullet **every frame** (~60/sec) because `update` runs every frame. `shoot_timer` is a simple cooldown: when space is held, subtract `dt`; `shoot()` refuses to fire while the timer is positive.
- `Shot.containers = (shots, updatable, drawable)` means the new bullet is automatically updated and drawn, and is also available for collision checks through the `shots` group.

---

## Collision and Splitting

In the main loop:

```python
for obj in asteroids:
    if obj.collides_with(player):
        log_event("player_hit")
        print("Game Over!")
        sys.exit()

    for shot in shots:
        if obj.collides_with(shot):
            log_event("asteroid_shot")
            shot.kill()
            obj.split()
```

- `collides_with` compares distance between centers to the sum of radii (circle-circle collision).
- `sprite.kill()` removes the sprite from **all** groups it belongs to. That's how a shot disappears: it's removed from `shots`, `updatable`, and `drawable` in one call. No manual list cleanup needed.
- `split()` on the asteroid:

```python
def split(self):
    self.kill()
    if self.radius <= ASTEROID_MIN_RADIUS:
        return                        # smallest asteroids just vanish
    log_event("asteroid_split")
    rand_angle = random.uniform(20, 50)
    first_new_vector = self.velocity.rotate(rand_angle)
    second_new_vector = self.velocity.rotate(-rand_angle)
    new_radius = self.radius - ASTEROID_MIN_RADIUS
    a1 = Asteroid(self.position.x, self.position.y, new_radius)
    a2 = Asteroid(self.position.x, self.position.y, new_radius)
    a1.velocity = first_new_vector * 1.2
    a2.velocity = second_new_vector * 1.2
```

Two smaller asteroids are created at the same spot, with velocities rotated ±20–50° from the parent and sped up by 1.2×. They auto-register into the groups, so they're immediately updated, drawn, and collidable.

---

## Constants Cheat Sheet

| Constant | Value | Used for |
|---|---|---|
| `SCREEN_WIDTH` / `SCREEN_HEIGHT` | 1280 / 720 | window size, edge spawn points |
| `PLAYER_RADIUS` | 20 | player collision + drawing |
| `LINE_WIDTH` | 2 | outline thickness |
| `PLAYER_TURN_SPEED` | 300 | degrees per second |
| `PLAYER_SPEED` | 200 | pixels per second |
| `PLAYER_SHOOT_SPEED` | 500 | bullet pixels per second |
| `PLAYER_SHOOT_COOLDOWN_SECONDS` | 0.3 | minimum time between shots |
| `SHOT_RADIUS` | 5 | bullet size |
| `ASTEROID_MIN_RADIUS` | 20 | smallest asteroid / split step |
| `ASTEROID_KINDS` | 3 | sizes: 20, 40, 60 |
| `ASTEROID_MAX_RADIUS` | 60 | spawn just outside the screen |
| `ASTEROID_SPAWN_RATE_SECONDS` | 0.8 | time between spawns |

---

## Checklist: Adding a New Game Object Type

1. Create the class, inherit from `CircleShape` (or `pygame.sprite.Sprite` if it's not a shape).
2. Implement `__init__`, `draw`, and `update` (the base ones are empty stubs).
3. In `main.py`, create any extra group you need (e.g. `shots`).
4. Set `YourClass.containers = (extra_group, updatable, drawable)` **before** creating instances.
5. If it moves, set `velocity` and use `position += velocity * dt` in `update`.
6. Use `sprite.kill()` to remove it from all groups at once.

---

## Gotchas I Hit

- **`containers` must be set before instantiation.** It's a class attribute; existing objects are not retroactively added.
- **`dt` starts at `0.0`.** The first frame doesn't move anything because `dt` is assigned at the end of the loop.
- **`tick(60)` is a cap, not a promise.** Heavy frames run below 60 FPS.
- **Held keys belong in `update` (`key.get_pressed()`), not in the event loop.** Events are one-shot.
- **`draw` must not move anything.** If an object is invisible or frozen, check its `draw`/`update` overrides — inheriting a `pass` stub does nothing.
- **`kill()` is the clean removal.** It detaches the sprite from every group it was added to.
- **A shot without a cooldown fires every frame** while space is held, because `update` runs ~60 times per second.
