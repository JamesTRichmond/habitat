# habitat

A robot who lives on a web page. He walks, jumps, gets picked up and thrown, and when you leave him alone long enough he wanders off and finds something to do.

**[Live demo](https://jamestrichmond.github.io/habitat/)** · **[Seen in the wild](https://jamestrichmond.com/#habitat)**

No dependencies. No build step. One `<script type="module">` and an element to live in.

---

## Use

```js
import { createHabitat } from "./src/habitat.js";

const habitat = createHabitat(document.querySelector("#habitat"));
```

```html
<div id="habitat" style="aspect-ratio: 12/5; min-height: 260px"></div>
```

Give the host element a size and he fills it. He brings his own SVG, his own gradients, and his own colors, so nothing is asked of the page around him.

| Control | |
|---|---|
| `←` `→` or `A` `D` | walk |
| `space`, `↑`, `W` | jump |
| drag | pick him up and throw him |
| click | say hi |
| ignore him | he finds his own entertainment |

Keyboard input is scoped to the habitat, so arrow keys never steal the page's scroll.

## Options

```js
createHabitat(host, {
  theme: { accent: "#ff5f7e", visor: "#8be9fd" },  // partial override
  activities: { ...ACTIVITIES, wave },              // add your own
  scene: "workshop",                                // which one he opens in
  autonomy: true,                                   // let him wander off
  boredom: 11,                                      // seconds alone before he does
  status: true,                                     // the little state label
  keyboard: true,
  pointer: true,
  onState(name) { /* "idle", "walk", "cook", … */ },
});
```

The returned handle:

```js
habitat.command("ride");   // any activity name, or "idle" to come home
habitat.state;             // what he's doing right now
habitat.activities;        // the names he knows
habitat.pause();
habitat.resume();
habitat.destroy();         // takes its listeners and its rAF loop with it
```

## How he works

He is not a sprite sheet. He is a jointed SVG rig — two segments per limb, so knees and elbows actually bend — and every frame a state machine writes joint angles into a pose object, which is then applied to the DOM.

The life comes from four cheap things:

- **Gravity and ground contact.** Real integration, not a canned arc, so a throw and a hop land differently.
- **A damped spring on the antenna.** It lags going into a move and overshoots coming out of one. This is the single highest-value line of code in the project.
- **Squash and stretch on landing**, scaled by impact speed.
- **A pupil that chases your cursor**, with a blink on a random timer.

None of it is expensive. All of it is the difference between a shape being moved around and a thing that appears to be moving itself.

**Free states** — idle, walk, air, drag, greet, travel — run physics. **Activities** take the transform over and script it. That split is the whole architecture: it means an activity can do something physics could never express (lying under a motorcycle) without leaking special cases into the simulation.

## Adding something for him to do

An activity is a pose function and a mark to stand on. The entire "he waves at the wall":

```js
wave: {
  scene: "workshop", mark: 500, duration: 8, status: "waving",
  pose(p, t) {
    p.x = 500;
    p.armF = -150 + Math.sin(t * 9) * 22;
    p.elbowF = 26;
    p.headR = -5;
  },
}
```

`p` is the pose: `x`, `y`, `face`, `lean`, `bob`, `headR`, `armF/armB`, `elbowF/elbowB`, `legF/legB`, `kneeF/kneeB`, `visorOpen`, `coreDim`, `noShadow`. It also carries helpers — `p.fx.emit(kind, x, y)` for particles, `p.prop(name, {...})` to move a scene fixture, `p.sceneEl(class)` to reach into the current scene, and `p.lookAt(x, y)`.

Pass your map in as `activities` and he'll fold it into his rotation.

### The angle convention, which will bite you

Angles are degrees from hanging straight down. A limb at angle `t` points along `(−sin t, cos t)`.

When the whole rig is rotated −90° to lie him down, that becomes `(cos t, sin t)` on stage — so **0 runs toward his feet, −90 is straight up, and positive angles dip toward the floor.** Writing a lying-down pose as though 0 were still "down" is how you put a robot's arm through the ground, which happened here more than once.

## Scenes

Four ship with him: `workshop` (home), `kitchen`, `road`, `mountain`. Each has up to three bands — `art` behind him, `mid` behind him but in front of the wall, and `fg` in front of him, so he can stand behind a kitchen counter the way a person would. All of them are built once at boot and cross-faded by opacity, so switching costs nothing.

## Performance

One `requestAnimationFrame` loop. It stops when the tab is hidden and when the habitat scrolls out of view, so an idle page costs nothing. Delta time is clamped, so a slow frame can't teleport him across the room. Everything animated is a transform on about a dozen SVG nodes.

`prefers-reduced-motion` turns off his autonomy — he stays put and stays drivable.

## Files

```
src/
  habitat.js     public entry: builds the SVG, applies the theme, owns the loop
  robot.js       the simulation: physics, state machine, pose application
  rig.js         joint pivots and the SVG body that hangs off them
  activities.js  the five things he does when nobody's driving
  scenes.js      the four places he can be, and his props
```

## License

MIT — see [LICENSE](LICENSE).

Built by [James Richmond](https://jamestrichmond.com).
