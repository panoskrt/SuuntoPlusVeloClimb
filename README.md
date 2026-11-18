# Velo Climb

A SuuntoPlus feature app for cycling that displays live climbing metrics, an animated climb-category gauge, live heart rate, and adds climb statistics to the workout summary.

## Current features

- Smoothed live gradient based on vertical speed and ground speed.
- Rolling 30-second VAM (vertical ascent rate).
- Session-average VAM over moving time, for the workout summary.
- Maximum smoothed gradient for the current session.
- Total ascent from Suunto altitude data.
- Running average gradient from valid moving samples.
- Eight climb categories from Descent through HC+.
- An animated 8-segment corona gauge with a needle pointing at the current climb category.
- Live heart rate reading on the climb screen, with the heart icon coloured by the user's heart rate zone.
- Two watch screens that can be switched with the watch buttons.
- Workout summary outputs for VAM, average gradient, and maximum gradient.

## Telemetry inputs

The app declares these subscribed inputs in `manifest.json`:

| Input | Source | Usage |
|-------|--------|-------|
| `speed` | `Activity/Current/Speed` | Ground speed for gradient calculation |
| `vSpeed` | `Fusion/Altitude/VerticalSpeed` | Vertical speed for gradient calculation |
| `ascent` | `Fusion/Altitude/Ascent` | Total ascent, rolling VAM and average VAM |

The templates also read these platform resources directly:

- `Fusion/Altitude/AscentTime` for climb duration.
- `Navigation/Routes/NavigatedRoute/RemainAscent` for remaining route ascent.
- `Activity/Move/-1/Heartrate/Current` for the live heart rate reading and the heart icon colour on the climb screen.
- `Activity/Zones/Heartrate/Zone1/Limit`..`Zone5/Limit` for the heart icon colour.

## Calculations

### Gradient

When ground speed is greater than `0.3 m/s`, the instantaneous gradient is calculated as:

```text
gradient = (vertical speed / ground speed) * 100
```

The raw value is limited to `-60%` through `+60%`, then smoothed with an exponential moving average using `alpha = 0.2`. When the speed threshold is not met, the previous smoothed value is retained.

The app tracks the maximum smoothed gradient and calculates average gradient as the mean of smoothed-gradient samples collected while the speed threshold is met. State is initialized when the app loads and is held for the current app session.

### VAM

`evaluate()` is expected to run approximately once per second. The app stores up to 31 ascent samples and calculates the current ascent rate from the oldest and newest values in that rolling window:

```text
VAM = (latest ascent - oldest ascent) / elapsed sample intervals
```

The output uses Suunto's `VerticalSpeedMountain_Fourdigits` format.

If the ascent input is not available yet, VAM stays at `0` and total ascent keeps its last known value (initially `0`). A missed ascent sample repeats the last known value, so the window still covers the right number of seconds.

### Average VAM

The workout summary's `Avg VAM` is a session average, published as the `avgVam` output:

```text
average VAM = ascent gained while moving / seconds spent moving
```

"Moving" uses the same `0.3 m/s` speed threshold as the gradient, so time spent stopped (and any barometer drift during it) does not affect the average.

## Climb categories

Categories are based on the smoothed gradient, including a dedicated category for descents.

| Output value | Gradient | Label | Color |
|--------------|----------|-------|-------|
| `0` | `< 0%` | Descent | Blue |
| `1` | `0%` to `< 2%` | Flat | Gray |
| `2` | `2%` to `< 5%` | Cat4 | Green |
| `3` | `5%` to `< 7%` | Cat3 | Blue |
| `4` | `7%` to `< 8%` | Cat2 | Yellow |
| `5` | `8%` to `< 10%` | Cat1 | Orange |
| `6` | `10%` to `< 12%` | HC | Red |
| `7` | `>= 12%` | HC+ | Dark red |

On the climb screen, the current category also drives an animated corona gauge: an 8-segment ring drawn on a full-screen canvas, with a needle pointing at the active segment and the active segment highlighted in its category color. The gauge redraws whenever the category output changes.

## Heart rate zones

The climb screen (`t.html` / `t_p.html`) colours the heart icon by heart rate zone:

| Zone | Meaning | Icon colour |
| --- | --- | --- |
| none | No heart rate yet, zone limits not available, or below a Zone 1 limit (when one is set) | White |
| 1 | Zone 1 | Gray |
| 2 | Zone 2 | Blue |
| 3 | Zone 3 | Green |
| 4 | Zone 4 | Orange |
| 5 | Zone 5 | Red |

The zone is worked out in the template, not in `main.js`, and none of the heart rate resources are manifest inputs. With the heart rate and zone limits declared as manifest inputs, the watch deactivated the whole app when no HR belt was paired (wrist HR). The likely cause is a manifest input the watch could not supply in that setup; a template subscription that never delivers just leaves its value empty.

The limits are the user's zone limits, `Activity/Zones/Heartrate/ZoneN/Limit`. Each is the lower boundary of zone N; Zone 1 normally has none, so anything below the Zone 2 limit counts as Zone 1. If the Zone 2-5 limits are not all available the icon stays white.

Heart rate and limits are normalised to bpm (anything under 10 is treated as Hz), so the comparison works whichever unit a value arrives in.

The simulator defaults to Zone 2-5 limits of about 139, 149, 158 and 168 bpm. Behaviour on a watch, with an HR belt and with wrist HR, still needs confirming.

## Watch screens

### `t.html` - Climb screen

Displays:

- Current gradient, formatted as a percentage.
- Current rolling VAM.
- Live heart rate.
- Maximum gradient.
- Total ascent.
- Current category label with its category color, plus the animated corona gauge and needle described above.
- Separate large-display and small/medium-display layouts.

Pressing the watch's lower/down button sends an app event and switches to `t2.html`.

While the exercise is paused the app shows `t_p.html` instead: the same screen without the down-button binding, so the down button keeps its system function (end/save the activity). Any layout change to `t.html` must be mirrored in `t_p.html`.

### `t2.html` - Profile screen

Displays:

- Climb duration from `Fusion/Altitude/AscentTime`.
- Average gradient for the current app session.
- Remaining route ascent when navigation data is available.
- Separate large-display and small/medium-display layouts.

Pressing the watch's upper/up button sends an app event and switches back to `t.html`.

The altitude-profile graph markup is present in the file but currently commented out, so no graph is rendered.

## Workout summary

`main.js` exposes these summary outputs:

| ID | Name | Format | Value |
|----|------|--------|-------|
| `vam` | Avg VAM | `VerticalSpeedMountain_Fourdigits` | Session-average VAM over moving time (`avgVam` output) |
| `gradient` | Avg gradient | `Percentage_Fourdigits` | Running average gradient |
| `maxGradient` | Max gradient | `Percentage_Fourdigits` | Maximum smoothed gradient |

## Project structure

- `manifest.json` - App metadata, declared telemetry inputs and outputs, registered templates, and workout usage.
- `main.js` - State initialization, gradient and VAM calculations, category classification, screen switching, and summary outputs.
- `t.html` - Climb screen template, including the corona gauge canvas drawing logic.
- `t_p.html` - Copy of `t.html` without the down-button binding, shown while the exercise is paused.
- `t2.html` - Profile screen template.
- `cyclin01-l.fea`, `cyclin01-m.fea`, `cyclin01-n.fea`, `cyclin01-o.fea`, `cyclin01-q.fea`, `cyclin01-s.fea` - Compiled ZIP-based Suunto resources for the supported device/display variants (gitignored build output, not checked in).
- `LICENSE` - GPL-2.0 license text.

There is currently no package manager configuration, build script, automated test suite, or simulator configuration in this repository.

## App metadata

- **Name:** Velo Climb
- **Version:** 1.0
- **Type:** SuuntoPlus feature app
- **Usage:** Workout
- **Author:** Panagiotis Kritikakos

## License

GPL-2.0 - see [LICENSE](LICENSE).
