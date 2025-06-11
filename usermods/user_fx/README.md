# Usermod user FX

This Usermod is a common place to put various user's private LED Effects.  It gives you a way to load in your own custom WLED effects, or to load in depracated WLED effects that you want to bring back--all without having to mess with the core WLED source code.

Multiple Effects can be specified inside this single usermod, as we will illustrate below.  You will be able to define them with custom names, sliders, etc. as with any other Effect.



## How The Usermod Works

The `user_fx.cpp` file can be broken down into four main parts:
* **static effect definition** - This is a static LED setting that is displayed if an effect fails to initialize.
* **User FX function definition(s)** - This area is where you place the FX code for all of the custom effects you want to use.  This mainly includes the FX code and the static variable containing the [metadata string](https://kno.wled.ge/interfaces/json-api/#effect-metadata). 
* **Usermod Class definition(s)** - The class definition defines the blueprint from which all your custom Effects (or any usermod, for that matter) are created.
* **Usermod registration** - All usermods have to be registered so that they are able to be compiled into your binary.

We will go into greater detail on how custom effects work in the usermod and how to go abour creating your own in the section below.


## Basic Syntax for WLED Effect Creation
Below are some helpful variables and functions to know as you start your journey towards WLED effect creation:
* **SEGMENT.intensity / speed / custom1 etc.** -> This syntax helps define the UI sliders that can make certain elements of your running code editable by the user.  (These can be controlled by the API as well.)
* **SEGENV.call / step / aux0 / aux1** - these are useful counters for you to use in your code.  
* **SEGLEN / SEG_W / SEG_H** -> These variables help define the length and width of your LED strip/matrix.  They can be changed at any time if the user sets new segment size(s).
* **SEGCOLOR(x) / SEGPALETTE** for user colors from UI (and how it relates to meta string)
* explaining available functions for pixel drawing (the ones defined in FX_2Dfcn and FX_fcn, like move(), blur() etc.)
* **SEGMENT.color_from_palette()** function (which should be favoured over ColorFromPalette())


## Understanding 2D WLED Effects

In this section we give some advice to those who are new to WLED Effect creation.  We will illustrate how to load in multiple Effects using this single usermod, and we will do a deep dive into the anatomy of a 1D Effect as well as a 2D Effect.
(Special thanks to @mryndzionek for offering this "Diffusion Fire" 2D Effect for this tutorial.)

### Imports
The first line of the code imports the [wled.h](https://github.com/wled/WLED/blob/main/wled00/wled.h) file into this module.  This file handles all other imports and it has all the global variable declarations you'd need for your Effects.

```
#include "wled.h"
```

### Static Effect Definition
The next code block is the `mode_static` definition.  This is usually left as `SEGMENT.fill(SEGCOLOR(0));` to leave all pixels off if the effect fails to load, but in theory one could use this as a 'fallback effect' to take on a different behavior, such as...
TODO

### User Effect Definitions
Pre-loaded in this template is an example 2D Effect called "Diffusion Fire".  (This is the name that would be shown in the UI once the binary is compiled and run on your device, as defined in the metadata string.)
The effect starts off by checking to see if the segment that the effect is being applied to is a 2D Matrix, and if it is not, then it returns the static effect which displays no pattern:
```
if (!strip.isMatrix || !SEGMENT.is2D()) 
return mode_static();  // not a 2D set-up 
```
The next code block contains several constant variable definitions which essentially serve to extract the dimensions of the user's 2D matrix and allow WLED to interpret the matrix as a 1D coordinate system (WLED must do this for all 2D animations):
```
const int cols = SEG_W;
const int rows = SEG_H;
const auto XY = [&](int x, int y) { return x + y * cols; };
```
* The first line assigns the number of columns (width) in the active segment to cols.
  * SEG_W is a macro defined in WLED that expands to SEGMENT.width().  This value is the width of your 2D matrix segment, used to traverse the matrix correctly.
* Next, we assign the number of rows (height) in the segment to rows.
  * SEG_H is a macro for SEGMENT.height(). Combined with cols, this allows pixel addressing in 2D (x, y) space.
* The third line declares a lambda function named XY to convert (x, y) matrix coordinates into a 1D index in the LED array.  This assumes row-major order (left to right, top to bottom).
  * WLED internally treats the LED strip as a 1D array, so effects must translate 2D coordinates into 1D indices. This lambda helps with that.

The next lines of code further the setup process by defining variables that allow the effect's settings to be configurable using the UI sliders (or alternatively, through API calls):
```
const uint8_t refresh_hz = map(SEGMENT.speed, 0, 255, 20, 80);
const unsigned refresh_ms = 1000 / refresh_hz;
const int16_t diffusion = map(SEGMENT.custom1, 0, 255, 0, 100);
const uint8_t spark_rate = SEGMENT.intensity;
const uint8_t turbulence = SEGMENT.custom2;
```
* The first line maps the SEGMENT.speed (user-controllable parameter from 0–255) to a value between 20 and 80 Hz.
  * This determines how often the effect should refresh per second (Higher speed = more frames per second).
* Next we convert refresh rate from Hz to milliseconds. (It’s easier to schedule animation updates in WLED using elapsed time in milliseconds.)
  * This value is used to time when to update the effect.
* The third line utilizes the `custom1` control (0–255 range, usually exposed via sliders) to define the diffusion rate, mapped to 0–100.
  * This controls how much "heat" spreads to neighboring pixels — more diffusion = smoother flame spread.
* Next we assign `SEGMENT.intensity` (user input 0–255) to a variable named `spark_rate`.
  * This controls how frequently new "spark" pixels appear at the bottom of the matrix.
  * A higher value means more frequent ignition of flame points.
* The final line stores the user-defined `custom2` value to a variable called `turbulence`.
  * This is used to introduce randomness in spark generation or flow — more turbulence means more chaotic behavior.

Next we will look at some lines of code that handle memory allocation and effect initialization:

```
unsigned dataSize = SEGMENT.length();
```
* This part calculates how much memory we need to represent per-pixel state.
  * `SEGMENT.length()` returns the total number of LEDs in the current segment (i.e., cols * rows in a matrix).
  * This fire effect models heat values per pixel (not just colors), so we need persistent storage — one uint8_t per pixel — for the entire effect.

```
if (!SEGENV.allocateData(dataSize))
return mode_static(); // allocation failed
```
* This section allocates a persistent data buffer tied to the segment environment (`SEGENV.data`).
* The syntax `SEGENV.allocateData(n)` requests a buffer of size n bytes (1 byte per pixel here).
* If allocation fails (e.g., out of memory), it returns false, and the effect can’t proceed.
* It calls previously defined `mode_static()` fallback effect, which just fills the segment with a static color.  We need to do this because WLED needs a fail-safe behavior if a custom effect can't run properly due to memory constraints.


The next lines of code clear the LEDs and initialize timing:
```
if (SEGENV.call == 0) {
  SEGMENT.fill(BLACK);
  SEGENV.step = 0;
}
```
* The first line checks whether this is the first time the effect is being run; `SEGENV.call` is a counter for how many times this effect function has been invoked since it started.
* If `SEGENV.call` equals 0 (which it does on the very first call, making it useful for initialization), then it clears the LED segment by filling it with black (turns off all LEDs).
* This gives a clean starting point for the fire animation.
* It also initializes `SEGENV.step`, a timing marker, to 0.  This value is later used as a timestamp to control when the next animation frame should occur (based on elapsed time).

The next block of code is where the animation update logic starts to kick in:
```
if ((strip.now - SEGENV.step) >= refresh_ms) {
  uint8_t tmp_row[cols];
  SEGENV.step = strip.now;
  // scroll up
  for (unsigned y = 1; y < rows; y++)
    for (unsigned x = 0; x < cols; x++) {
      unsigned src = XY(x, y);
      unsigned dst = XY(x, y - 1);
      SEGMENT.data[dst] = SEGMENT.data[src];
    }
```
* The first line checks if it's time to update the effect frame.  `strip.now` is the current timestamp in milliseconds; `SEGENV.step` is the last update time (set during initialization or previous frame).  `refresh_ms` is how long to wait between frames, computed earlier based on SEGMENT.speed.
* The conditional statement in the first line fo code ensures the effect updates on a fixed interval — e.g., every 20 ms for 50 Hz.
* The second line of code declares a temporary row buffer for intermediate diffusion results that is one byte per column (horizontal position), so this buffer holds one row's worth of heat values.
* You'll see later that it writes results here before updating `SEGMENT.data`.
  * Note that this is declared on the stack each frame. Since the number of columns is typically small (e.g. ≤ 16), it's efficient.

Now we get to the spark generation portion, where new bursts of heat appear at the bottom of the matrix:
```
if (hw_random8() > turbulence) {
  // create new sparks at bottom row
  for (unsigned x = 0; x < cols; x++) {
    uint8_t p = hw_random8();
    if (p < spark_rate) {
      unsigned dst = XY(x, rows - 1);
      SEGMENT.data[dst] = 255;
    }
  }
}
```
* The first line randomizes whether we even attempt to spawn sparks this frame.
  * `hw_random8()` gives another random number between 0–255 using a fast hardware RNG, as described above.
  * `turbulence` is a user-controlled parameter (SEGMENT.custom2, set earlier).
  * Higher turbulence means this block is less likely to run (because `hw_random8()` is less likely to exceed a high threshold).
  * This adds randomness to when sparks appear — simulating natural flicker and chaotic fire.
* The next line loops over all columns in the bottom row (row `rows - 1`).
* Another random number, `p`, is used to probabilistically decide whether a spark appears at this (x, `rows-1`) position.
* Next is a conditional statement.  The lower spark_rate is, the fewer sparks will appear.
  * `spark_rate` comes from `SEGMENT.intensity` (0–255).
  * High intensity means more frequent ignition.
* `dst` calculates the destination index in the bottom row at column x.
* The final line here sets the heat at this pixel to maximum (255).
  * This simulates a fresh burst of flame, which will diffuse and move upward over time in subsequent frames.

Next we reach the first part of the core of the fire simulation, which is diffusion (how heat spreads to neighboring pixels):
```
// diffuse
for (unsigned y = 0; y < rows; y++) {
  for (unsigned x = 0; x < cols; x++) {
    unsigned v = SEGMENT.data[XY(x, y)];
    if (x > 0) {
      v += SEGMENT.data[XY(x - 1, y)];
    }
    if (x < (cols - 1)) {
      v += SEGMENT.data[XY(x + 1, y)];
    }
    tmp_row[x] = min(255, (int)(v * 100 / (300 + diffusion)));
  }
```
* This block of code starts by looping over each row from top to bottom.  (We will do diffusion for each pixel row.)
* Next we start an inner loop which iterates across each column in the current row.
* Starting with the current heat value of pixel (x, y) assigned `v`:
  * if there’s a pixel to the left, add its heat to the total.
  * If there’s a pixel to the right, add its heat as well.
  * So essentially, what the two `if` statements accomplish is: `v = center + left + right`.
* The final line of code applies diffusion smoothing:
  * The denominator controls how much the neighboring heat contributes. `300 + diffusion` means that with higher diffusion, you get more smoothing (since the sum is divided more).
  * The `v * 100` scales things before dividing (preserving some dynamic range).
  * `min(255, ...)` clamps the result to 8-bit range.
  * This entire line of code stores the smoothed heat into the temporary row buffer.

After calculating tmp_row, we now handle rendering the pixels by updating the actual segment data and turning 'heat' into visible colors:
```
  for (unsigned x = 0; x < cols; x++) {
    SEGMENT.data[XY(x, y)] = tmp_row[x];
    if (SEGMENT.check1) {
      uint32_t color = ColorFromPalette(SEGPALETTE, tmp_row[x], 255, LINEARBLEND_NOWRAP);
      SEGMENT.setPixelColorXY(x, y, color);
    } else {
      uint32_t color = SEGCOLOR(0);
      SEGMENT.setPixelColorXY(x, y, color_fade(color, tmp_row[x]));
    }
  }
}
```
* This next loop starts iterating over each row from top to bottom.  (We're now doing this for color-rendering for each pixel row.)
* Next we update the main segment data with the smoothed value for this pixel.
* The if statement creates a conditional rendering path — the user can toggle this.  If `check1` is enabled in the effect metadata, we use a color palette to display the flame.
* The next line converts the heat value (`tmp_row[x]`) into a `color` from the current palette with 255 brightness, and no wrapping in palette lookup.
  * This creates rich gradient flames (e.g., yellow → red → black).
* Finally we set the rendered color for the pixel (x, y).
  * This repeats for each pixel  in each row.
* If palette use is disabled, we fallback to fading a base color.
* `SEGCOLOR(0)` gets the first user-selected color for the segment.
* The final line of code fades that base color according to the heat value (acts as brightness multiplier).

The final piece of this custom effect returns the frame time:
```
}
return FRAMETIME;
}
```
* The first bracket closes the `if ((strip.now - SEGENV.step) >= refresh_ms)` block that started back on line 39.
  * It ensures that the fire simulation (scrolling, sparking, diffusion, rendering) only runs when enough time has passed since the last update.
* returning the frame time tells WLED how soon this effect wants to be called again.
  * `FRAMETIME` is a predefined macro in WLED, typically set to ~16ms, corresponding to ~60 FPS (frames per second).
  * Even though the effect logic itself controls when to update based on refresh_ms, WLED will still call this function at roughly FRAMETIME intervals to check whether an update is needed.
* ⚠️ Important: Because the actual frame logic is gated by strip.now - SEGENV.step, returning FRAMETIME here doesn’t cause excessive updates — it just keeps the engine responsive.
* The final bracket closes the `mode_diffusionfire()` function itself.

### The Metadata String
At the end of every effect is an important line of code called the **metadata string**.  
It defines how the effect is to be interacted with in the UI:
```
static const char _data_FX_MODE_DIFFUSIONFIRE[] PROGMEM = "Diffusion Fire@!,Spark rate,Diffusion Speed,Turbulence,,Use palette;;Color;;2;pal=35";
```
This string is passed into `strip.addEffect()` and parsed by WLED to determine how your effect appears and behaves in the UI. 
Let's break it down field by field, following WLED's metadata format.
The string format WLED uses is semicolon-separated sections, with commas used for control labels.
Let’s split this into logical sections (delimited by semicolons ;):
**TODO**

### The UserFxUsermod Class

The `UserFxUsermod` class registers the `mode_diffusionfire` effect with WLED. This section starts right after the effect function and metadata string, and is responsible for making the effect usable in the WLED interface:
```
class UserFxUsermod : public Usermod {
 private:
 public:
  void setup() override {
    strip.addEffect(255, &mode_diffusionfire, _data_FX_MODE_DIFFUSIONFIRE);

    ////////////////////////////////////////
    //  add your effect function(s) here  //
    ////////////////////////////////////////

    // use id=255 for all custom user FX (the final id is assigned when adding the effect)

    // strip.addEffect(255, &mode_your_effect, _data_FX_MODE_YOUR_EFFECT);
    // strip.addEffect(255, &mode_your_effect2, _data_FX_MODE_YOUR_EFFECT2);
    // strip.addEffect(255, &mode_your_effect3, _data_FX_MODE_YOUR_EFFECT3);
  }
  void loop() override {} // nothing to do in the loop
  uint16_t getId() override { return USERMOD_ID_USER_FX; }
};
```
* The first line declares a new class called UserFxUsermod. It inherits from `Usermod`, which is the base class WLED uses for any pluggable user-defined modules.
  * This makes UserFxUsermod a valid WLED extension that can hook into `setup()`, `loop()`, and other lifecycle events.
* The `void setup()` function runs once when WLED initializes the usermod.
  * It's where you should register your effects, initialize hardware, or do any other setup logic.
  * `override` ensures that this matches the Usermod base class definition.
* The `strip.addEffect` line is an important one that registers the custom effect so WLED knows about it.
  * 255: Temporary ID — WLED will assign a unique ID automatically.  (**Create all custom effects with the 255 ID.**)
  * `&mode_diffusionfire`: Pointer to the effect function.
  * `_data_FX_MODE_DIFFUSIONFIRE`: Metadata string stored in PROGMEM, describing the effect name and UI fields (like sliders).
  * After this, your custom effect shows up in the WLED effects list.
* The `loop()` function remains empty because this usermod doesn’t need to do anything continuously. WLED still calls this every main loop, but nothing is done here.
  * If your usermod had to respond to input or update state, you'd do it here.
* The last part returns a unique ID constant used to identify this usermod.
  * USERMOD_ID_USER_FX is defined in [const.h](https://github.com/wled/WLED/blob/main/wled00/const.h). WLED uses this for tracking, debugging, or referencing usermods internally.

The final part of this file handles instatiation and initialization:
```
static UserFxUsermod user_fx;
REGISTER_USERMOD(user_fx);
```
* The first line creates a single, global instance of your usermod class.
* The last line is a macro that tells WLED: “This is a valid usermod — load it during startup.”
  * WLED adds it to the list of active usermods, calls `setup()` and `loop()`, and lets it interact with the system.

## Understanding 1D WLED Effects

**TODO**

## Combining Multiple Effects in this Usermod

So now let's say that in addition to the custom "Diffusion Fire" effect detailed above, you also want to add the "Police" effect through this same Usermod file.  The Police effect was removed from WLED after v0.14.0, but through this guide, we will illustrate how to add it back into your WLED build, along with the custom Diffusion Fire effect:
* Navigate to the code for the Police effect [here](https://github.com/wled/WLED/blob/ef0f91d8d07c7aad2ed55f293f01cac3ef43976d/wled00/FX.cpp#L1226).
* Copy this code, and place it below the metadata string for Diffusion Fire.
* Add a [metadata string](https://kno.wled.ge/interfaces/json-api/#effect-metadata) for the effect.
* Register the effect using the `addEffect` function in the Usermod class.
* Compile the code.

## Compiling
[The complete guide to compiling WLED can be found here](https://kno.wled.ge/advanced/compiling-wled/), on the offical WLED documentation website.


## Change Log


