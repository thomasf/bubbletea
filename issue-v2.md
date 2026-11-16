## stop the ticker right after drawing unless the view changes every frame

> [!NOTE]
> On both test systems this matches `main` for views that change every frame
> or every other frame (within a few percent of its wakeups, or fewer), and
> wakes up less often in every other case measured: slower changes, bursts
> and irregular gaps.

In `main`(6b30ecf) the renderer stops its frame ticker after half a second of
frames without anything to draw. A program whose view changes more often than
that, like a clock, a system monitor or a progress display updating a few
times a second, never gets to stop it.

Stopping the ticker and starting it again costs about as many wakeups as one
tick with nothing to draw. The measurements below compare `main` with a
patched build that stops the ticker right after each frame it draws, unless
the view is changing every frame. While the last two frames drawn each
followed the one before, it keeps ticking, through one frame with nothing to
draw so that a change drifting across a frame boundary doesn't stop it.
Every change wakes the renderer, and a restarted ticker draws on the same
frame boundaries as before, so frames are drawn when they would have been
(the patch is at the end).

## Reproduction

A program that changes its view every `-every` at `-fps` frames a second
and does nothing else, for example `go run . -fps 60 -every 500ms`.
`-every` also takes a comma-separated list of gaps to cycle through, and
`-burst` and `-pause` alternate changing the view with pausing:

<details>
<summary>Example program</summary>

```go
package main

import (
	"flag"
	"fmt"
	"log"
	"strings"
	"time"

	tea "charm.land/bubbletea/v2"
)

type tick struct{}

type pause struct{}

type model struct {
	every        []time.Duration // cycled through
	burst, pause time.Duration   // burst 0: no pauses
	fps          int
	n            int
	started      time.Time // when the current burst started
}

func (m model) next() tea.Cmd {
	return tea.Tick(m.every[m.n%len(m.every)], func(time.Time) tea.Msg { return tick{} })
}

func (m model) Init() tea.Cmd { return func() tea.Msg { return pause{} } }

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
	switch msg := msg.(type) {
	case pause:
		m.started = time.Now()
		return m, m.next()
	case tick:
		m.n++
		if m.burst > 0 && time.Since(m.started) >= m.burst {
			return m, tea.Tick(m.pause, func(time.Time) tea.Msg { return pause{} })
		}
		return m, m.next()
	case tea.KeyPressMsg:
		if msg.String() == "q" {
			return m, tea.Quit
		}
	}
	return m, nil
}

func (m model) View() tea.View {
	return tea.NewView(fmt.Sprintf("every %v at %d fps: %d", m.every, m.fps, m.n))
}

func main() {
	every := []time.Duration{time.Second}
	flag.Func("every", "time between view changes; a comma-separated list is cycled through", func(s string) error {
		every = every[:0]
		for f := range strings.SplitSeq(s, ",") {
			d, err := time.ParseDuration(f)
			if err != nil {
				return err
			}
			every = append(every, d)
		}
		return nil
	})
	burst := flag.Duration("burst", 0, "change the view for this long, then pause; 0 changes it all the time")
	pause := flag.Duration("pause", 3*time.Second, "time between bursts")
	fps := flag.Int("fps", 60, "frame rate")
	flag.Parse()
	if _, err := tea.NewProgram(model{every: every, burst: *burst, pause: *pause, fps: *fps}, tea.WithFPS(*fps)).Run(); err != nil {
		log.Fatal(err)
	}
}
```

</details>

Test systems:

- desktop: AMD Ryzen 9 9950X3D (32 threads) / Linux 7.0.0-43-generic
- laptop: Intel Core Ultra 7 265U (14 cores), on AC, balanced power profile /
  Linux 7.0.0-42-generic

The same program at 30, 60 and 120 fps across change intervals, `main`
(6b30ecf) and patched side by side, mean of two 60 s runs.

Desktop:

| fps | view changes every | frames | CPU `main` | CPU patched | wakeups/s `main` | wakeups/s patched |
|----:|--------------------|-------:|-----------:|------------:|-----------------:|------------------:|
|  30 | 33 ms              |      1 | 159 ms/min |  141 ms/min |              333 |               313 |
|  30 | 67 ms              |      2 | 132 ms/min |  100 ms/min |              244 |               256 |
|  30 | 100 ms             |      3 |  91 ms/min |   83 ms/min |              209 |               171 |
|  30 | 133 ms             |      4 |  74 ms/min |   58 ms/min |              203 |               132 |
|  30 | 250 ms             |    7.5 |  74 ms/min |   38 ms/min |              184 |                70 |
|  30 | 500 ms             |     15 |  65 ms/min |   17 ms/min |              174 |                36 |
|  30 | 1 s                |     30 |  43 ms/min |    9 ms/min |               86 |                17 |
|  60 | 17 ms              |      1 | 232 ms/min |  242 ms/min |              607 |               595 |
|  60 | 33 ms              |      2 | 160 ms/min |  169 ms/min |              459 |               457 |
|  60 | 50 ms              |      3 | 166 ms/min |  120 ms/min |              401 |               309 |
|  60 | 67 ms              |      4 | 151 ms/min |   92 ms/min |              396 |               230 |
|  60 | 100 ms             |      6 | 138 ms/min |   72 ms/min |              367 |               158 |
|  60 | 250 ms             |     15 | 106 ms/min |   30 ms/min |              337 |                63 |
|  60 | 500 ms             |     30 |  93 ms/min |   19 ms/min |              330 |                32 |
|  60 | 1 s                |     60 |  55 ms/min |    7 ms/min |              174 |                17 |
| 120 | 8 ms               |      1 | 385 ms/min |  387 ms/min |              973 |               923 |
| 120 | 17 ms              |      2 | 263 ms/min |  260 ms/min |              744 |               734 |
| 120 | 25 ms              |      3 | 232 ms/min |  213 ms/min |              666 |               529 |
| 120 | 33 ms              |      4 | 225 ms/min |  167 ms/min |              632 |               400 |
| 120 | 100 ms             |     12 | 180 ms/min |   58 ms/min |              566 |               137 |
| 120 | 250 ms             |     30 | 154 ms/min |   28 ms/min |              544 |                56 |
| 120 | 500 ms             |     60 | 157 ms/min |   11 ms/min |              527 |                27 |
| 120 | 1 s                |    120 |  68 ms/min |    6 ms/min |              256 |                13 |

Laptop:

| fps | view changes every | frames | CPU `main`  | CPU patched | wakeups/s `main` | wakeups/s patched |
|----:|--------------------|-------:|------------:|------------:|-----------------:|------------------:|
|  30 | 33 ms              |      1 |  433 ms/min |  458 ms/min |              306 |               305 |
|  30 | 67 ms              |      2 |  296 ms/min |  279 ms/min |              230 |               240 |
|  30 | 100 ms             |      3 |  256 ms/min |  194 ms/min |              205 |               167 |
|  30 | 133 ms             |      4 |  257 ms/min |  153 ms/min |              197 |               123 |
|  30 | 250 ms             |    7.5 |  195 ms/min |  118 ms/min |              178 |                65 |
|  30 | 500 ms             |     15 |  160 ms/min |   42 ms/min |              171 |                34 |
|  30 | 1 s                |     30 |   88 ms/min |   25 ms/min |               91 |                17 |
|  60 | 17 ms              |      1 |  761 ms/min |  806 ms/min |              546 |               537 |
|  60 | 33 ms              |      2 |  540 ms/min |  529 ms/min |              432 |               426 |
|  60 | 50 ms              |      3 |  436 ms/min |  363 ms/min |              388 |               299 |
|  60 | 67 ms              |      4 |  427 ms/min |  291 ms/min |              376 |               224 |
|  60 | 100 ms             |      6 |  386 ms/min |  179 ms/min |              359 |               150 |
|  60 | 250 ms             |     15 |  311 ms/min |   73 ms/min |              337 |                61 |
|  60 | 500 ms             |     30 |  509 ms/min |   85 ms/min |              329 |                28 |
|  60 | 1 s                |     60 |  143 ms/min |   22 ms/min |              163 |                15 |
| 120 | 8 ms               |      1 | 1356 ms/min | 1307 ms/min |              905 |               936 |
| 120 | 17 ms              |      2 |  887 ms/min |  864 ms/min |              728 |               691 |
| 120 | 25 ms              |      3 |  730 ms/min |  658 ms/min |              654 |               530 |
| 120 | 33 ms              |      4 |  674 ms/min |  543 ms/min |              632 |               398 |
| 120 | 100 ms             |     12 |  547 ms/min |  169 ms/min |              559 |               135 |
| 120 | 250 ms             |     30 |  464 ms/min |   72 ms/min |              538 |                56 |
| 120 | 500 ms             |     60 |  531 ms/min |   38 ms/min |              528 |                28 |
| 120 | 1 s                |    120 |  219 ms/min |   19 ms/min |              254 |                14 |

When the view changes every frame or every other frame the two are within
a few percent; CPU in the fast rows varies by up to 20% between runs. From
every third frame on the patched build wakes up less often, and from 500 ms
on it uses less than a third of the CPU, on both systems.

## Bursts and irregular changes

The same program at 60 fps (a frame is 16.7 ms), side by side, mean of two
60 s runs.

Desktop:

|   | view changes                                     | CPU `main` | CPU patched | wakeups/s `main` | wakeups/s patched |
|---|--------------------------------------------------|-----------:|------------:|-----------------:|------------------:|
| A | every frame for 500 ms, then 3 s pause           |  54 ms/min |   42 ms/min |              131 |                89 |
| B | every other frame for 1 s, then 2 s pause        |  95 ms/min |   68 ms/min |              210 |               161 |
| C | every 100 ms for 2 s, then 3 s pause             |  75 ms/min |   31 ms/min |              181 |                65 |
| D | every frame for 300 ms, then 500 ms pause        | 178 ms/min |  127 ms/min |              428 |               245 |
| E | gaps alternating between 1 and 4 frames          | 184 ms/min |  145 ms/min |              438 |               309 |
| F | gaps of 1, 3, 1, 8, 2, 15, 1, 5 frames, repeated | 178 ms/min |  102 ms/min |              381 |               184 |

Laptop:

|   | view changes                                     | CPU `main` | CPU patched | wakeups/s `main` | wakeups/s patched |
|---|--------------------------------------------------|-----------:|------------:|-----------------:|------------------:|
| A | every frame for 500 ms, then 3 s pause           | 139 ms/min |  147 ms/min |              127 |                82 |
| B | every other frame for 1 s, then 2 s pause        | 235 ms/min |  166 ms/min |              204 |               146 |
| C | every 100 ms for 2 s, then 3 s pause             | 174 ms/min |   79 ms/min |              175 |                64 |
| D | every frame for 300 ms, then 500 ms pause        | 499 ms/min |  331 ms/min |              412 |               232 |
| E | gaps alternating between 1 and 4 frames          | 523 ms/min |  394 ms/min |              416 |               295 |
| F | gaps of 1, 3, 1, 8, 2, 15, 1, 5 frames, repeated | 426 ms/min |  275 ms/min |              371 |               178 |

The patched build wakes up less often in every scenario on both systems.
After a burst of changes every frame it stops ticking two frames after the
last one, instead of half a second later, and between changes that are
more than a frame apart it doesn't tick at all.

## Measurements in real programs

Side by side, mean of two 60 s runs:

| system  | program                                       | build            | wakeups/s | CPU        |
|---------|-----------------------------------------------|------------------|----------:|-----------:|
| desktop | system monitor, samples every 500 ms, 60 fps  | `main` (6b30ecf) |       240 | 246 ms/min |
|         |                                               | patched          |        22 | 152 ms/min |
|         | music player, 1 s clock while playing, 20 fps | `main` (6b30ecf) |        75 | 121 ms/min |
|         |                                               | patched          |        21 | 104 ms/min |
| laptop  | system monitor, samples every 500 ms, 60 fps  | `main` (6b30ecf) |       245 | 990 ms/min |
|         |                                               | patched          |        30 | 461 ms/min |

## Patched build

The patched build stops the ticker right after drawing, unless the view is
changing every frame:

<details>
<summary>Patch</summary>

```diff
--- a/tea.go
+++ b/tea.go
@@ -1519,8 +1519,6 @@ func (p *Program) Printf(template string, args ...any) {
 // startRenderer starts the renderer.
 func (p *Program) startRenderer() {
 	framerate := time.Second / time.Duration(p.fps)
-	// Stop ticking after about half a second of frames with nothing to draw.
-	idleFrames := max(p.fps/2, 1)
 
 	// Each run of the renderer gets its own ticker, so a goroutine that is
 	// still exiting can't stop the ticker of the next one.
@@ -1540,6 +1538,8 @@ func (p *Program) startRenderer() {
 			parked  bool      // whether the ticker is stopped
 			realign bool      // whether the ticker runs a shortened interval
 			last    time.Time // when the ticker last fired
+			drawn   time.Time // when a frame was last drawn
+			streak  int       // frames drawn in a row, each one frame after the one before
 		)
 		for {
 			select {
@@ -1563,10 +1563,19 @@ func (p *Program) startRenderer() {
 				}
 				_ = p.flush()
 				if drew, _ := p.renderer.flush(false); drew {
-					idle = 0
-				} else if idle++; idle >= idleFrames {
-					// Nothing is changing. Stop waking up every frame until
-					// there is something to draw, see [Program.wakeRenderer].
+					if !drawn.IsZero() && last.Sub(drawn) < 3*framerate/2 {
+						streak++
+					} else {
+						streak = 0
+					}
+					drawn, idle = last, 0
+				} else {
+					idle++
+				}
+				// A restart costs about as much as an empty tick: stop right
+				// after drawing, or while animating, on the second empty frame
+				// in a row. See [Program.wakeRenderer].
+				if animating := streak >= 2; !animating || idle > 1 {
 					ticker.Stop()
 					parked = true
 				}
```

</details>
