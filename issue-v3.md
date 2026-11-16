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
|  30 | 33 ms              |      1 | 172 ms/min |  164 ms/min |              320 |               331 |
|  30 | 67 ms              |      2 | 113 ms/min |  113 ms/min |              237 |               252 |
|  30 | 100 ms             |      3 |  89 ms/min |   90 ms/min |              208 |               175 |
|  30 | 133 ms             |      4 |  98 ms/min |   63 ms/min |              206 |               130 |
|  30 | 250 ms             |    7.5 |  68 ms/min |   28 ms/min |              182 |                70 |
|  30 | 500 ms             |     15 |  63 ms/min |   16 ms/min |              172 |                35 |
|  30 | 1 s                |     30 |  40 ms/min |    8 ms/min |               89 |                18 |
|  60 | 17 ms              |      1 | 245 ms/min |  240 ms/min |              611 |               600 |
|  60 | 33 ms              |      2 | 191 ms/min |  171 ms/min |              459 |               455 |
|  60 | 50 ms              |      3 | 174 ms/min |  113 ms/min |              409 |               303 |
|  60 | 67 ms              |      4 | 131 ms/min |  108 ms/min |              391 |               238 |
|  60 | 100 ms             |      6 | 142 ms/min |   60 ms/min |              366 |               153 |
|  60 | 250 ms             |     15 | 113 ms/min |   33 ms/min |              340 |                66 |
|  60 | 500 ms             |     30 | 116 ms/min |   16 ms/min |              333 |                31 |
|  60 | 1 s                |     60 |  49 ms/min |    9 ms/min |              161 |                16 |
| 120 | 8 ms               |      1 | 340 ms/min |  342 ms/min |              986 |               958 |
| 120 | 17 ms              |      2 | 248 ms/min |  248 ms/min |              755 |               725 |
| 120 | 25 ms              |      3 | 275 ms/min |  187 ms/min |              668 |               523 |
| 120 | 33 ms              |      4 | 199 ms/min |  157 ms/min |              635 |               395 |
| 120 | 100 ms             |     12 | 167 ms/min |   68 ms/min |              564 |               140 |
| 120 | 250 ms             |     30 | 139 ms/min |   22 ms/min |              541 |                55 |
| 120 | 500 ms             |     60 | 147 ms/min |   11 ms/min |              531 |                27 |
| 120 | 1 s                |    120 |  60 ms/min |    6 ms/min |              256 |                13 |

Laptop:

| fps | view changes every | frames | CPU `main`  | CPU patched | wakeups/s `main` | wakeups/s patched |
|----:|--------------------|-------:|------------:|------------:|-----------------:|------------------:|
|  30 | 33 ms              |      1 |  439 ms/min |  438 ms/min |              301 |               304 |
|  30 | 67 ms              |      2 |  276 ms/min |  272 ms/min |              231 |               241 |
|  30 | 100 ms             |      3 |  275 ms/min |  191 ms/min |              202 |               164 |
|  30 | 133 ms             |      4 |  221 ms/min |  152 ms/min |              200 |               124 |
|  30 | 250 ms             |    7.5 |  186 ms/min |   75 ms/min |              180 |                67 |
|  30 | 500 ms             |     15 |  161 ms/min |   54 ms/min |              170 |                32 |
|  30 | 1 s                |     30 |   91 ms/min |   23 ms/min |               89 |                17 |
|  60 | 17 ms              |      1 |  726 ms/min |  758 ms/min |              547 |               552 |
|  60 | 33 ms              |      2 |  493 ms/min |  493 ms/min |              433 |               428 |
|  60 | 50 ms              |      3 |  428 ms/min |  342 ms/min |              394 |               302 |
|  60 | 67 ms              |      4 |  365 ms/min |  284 ms/min |              383 |               227 |
|  60 | 100 ms             |      6 |  343 ms/min |  185 ms/min |              362 |               151 |
|  60 | 250 ms             |     15 |  304 ms/min |   77 ms/min |              336 |                61 |
|  60 | 500 ms             |     30 |  273 ms/min |   36 ms/min |              327 |                30 |
|  60 | 1 s                |     60 |  139 ms/min |   20 ms/min |              162 |                15 |
| 120 | 8 ms               |      1 | 1245 ms/min | 1254 ms/min |              902 |               900 |
| 120 | 17 ms              |      2 |  826 ms/min |  818 ms/min |              730 |               695 |
| 120 | 25 ms              |      3 |  658 ms/min |  601 ms/min |              653 |               528 |
| 120 | 33 ms              |      4 |  663 ms/min |  470 ms/min |              631 |               405 |
| 120 | 100 ms             |     12 |  509 ms/min |  159 ms/min |              559 |               136 |
| 120 | 250 ms             |     30 |  425 ms/min |   64 ms/min |              539 |                55 |
| 120 | 500 ms             |     60 |  612 ms/min |   38 ms/min |              528 |                28 |
| 120 | 1 s                |    120 |  213 ms/min |   20 ms/min |              257 |                14 |

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
| A | every frame for 500 ms, then 3 s pause           |  80 ms/min |   46 ms/min |              135 |                92 |
| B | every other frame for 1 s, then 2 s pause        |  97 ms/min |   74 ms/min |              217 |               162 |
| C | every 100 ms for 2 s, then 3 s pause             |  75 ms/min |   31 ms/min |              177 |                67 |
| D | every frame for 300 ms, then 500 ms pause        | 201 ms/min |  142 ms/min |              433 |               256 |
| E | gaps alternating between 1 and 4 frames          | 219 ms/min |  151 ms/min |              440 |               314 |
| F | gaps of 1, 3, 1, 8, 2, 15, 1, 5 frames, repeated | 176 ms/min |  118 ms/min |              381 |               187 |

Laptop:

|   | view changes                                     | CPU `main` | CPU patched | wakeups/s `main` | wakeups/s patched |
|---|--------------------------------------------------|-----------:|------------:|-----------------:|------------------:|
| A | every frame for 500 ms, then 3 s pause           | 162 ms/min |  134 ms/min |              125 |                84 |
| B | every other frame for 1 s, then 2 s pause        | 217 ms/min |  176 ms/min |              205 |               149 |
| C | every 100 ms for 2 s, then 3 s pause             | 211 ms/min |   98 ms/min |              175 |                62 |
| D | every frame for 300 ms, then 500 ms pause        | 487 ms/min |  330 ms/min |              415 |               239 |
| E | gaps alternating between 1 and 4 frames          | 522 ms/min |  405 ms/min |              412 |               293 |
| F | gaps of 1, 3, 1, 8, 2, 15, 1, 5 frames, repeated | 482 ms/min |  290 ms/min |              370 |               175 |

The patched build wakes up less often in every scenario on both systems.
After a burst of changes every frame it stops ticking two frames after the
last one, instead of half a second later, and between changes that are
more than a frame apart it doesn't tick at all.

## Measurements in real programs

Side by side, mean of two 60 s runs:

| system  | program                                       | build            | wakeups/s | CPU        |
|---------|-----------------------------------------------|------------------|----------:|-----------:|
| desktop | system monitor, samples every 500 ms, 60 fps  | `main` (6b30ecf) |       240 | 279 ms/min |
|         |                                               | patched          |        22 | 170 ms/min |
|         | music player, 1 s clock while playing, 20 fps | `main` (6b30ecf) |        78 | 139 ms/min |
|         |                                               | patched          |        21 | 107 ms/min |
| laptop  | system monitor, samples every 500 ms, 60 fps  | `main` (6b30ecf) |       246 | 972 ms/min |
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
+					} else if streak < 2 || idle > 1 {
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
