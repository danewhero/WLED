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


## Understanding WLED Effects

In this section we give some advice to those who are new to WLED Effect creation.  We will illustrate how to load in multiple Effects using this single usermod, and we will do a deep dive into the anatomy of a 1D Effect as well as a 2D Effect.

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
The next code block contains several constant variable definitions which essentially serve to extract the dimensions of the user's 2D matrix and allow WLED to interpret the matrix as a 1D coordinate system (WLED must do this for all 2D animations). We will walk through each line one by one:
* `const int cols = SEG_W;` -  Assigns the number of columns (width) in the active segment to cols.  SEG_W is a macro defined in WLED that expands to SEGMENT.width().  This value is the width of your 2D matrix segment, used to traverse the matrix correctly.
* `const int rows = SEG_H;` - Assigns the number of rows (height) in the segment to rows.  SEG_H is a macro for SEGMENT.height(). Combined with cols, this allows pixel addressing in 2D (x, y) space.
* `const auto XY = [&](int x, int y) { return x + y * cols; };` - Declares a lambda function named XY to convert (x, y) matrix coordinates into a 1D index in the LED array.  This assumes row-major order (left to right, top to bottom).  WLED internally treats the LED strip as a 1D array, so effects must translate 2D coordinates into 1D indices. This lambda helps with that.

The next lines of code further the setup process by defining variables that allow the effect's settings to be configurable using the UI sliders (or alternatively, through API calls):

* `const uint8_t refresh_hz = map(SEGMENT.speed, 0, 255, 20, 80);` - Maps the SEGMENT.speed (user-controllable parameter from 0–255) to a value between 20 and 80 Hz.  This determines how often the effect should refresh per second (Higher speed = more frames per second).
* `const unsigned refresh_ms = 1000 / refresh_hz;` - Converts refresh rate from Hz to milliseconds. It’s easier to schedule animation updates in WLED using elapsed time in milliseconds. This value is used to time when to update the effect.
* `const int16_t diffusion = map(SEGMENT.custom1, 0, 255, 0, 100);` - Uses the custom1 control (0–255 range, usually exposed via sliders) to define the diffusion rate, mapped to 0–100.  This controls how much "heat" spreads to neighboring pixels — more diffusion = smoother flame spread.
* `const uint8_t spark_rate = SEGMENT.intensity;` - Assigns SEGMENT.intensity (user input 0–255) to a variable named spark_rate.  Controls how frequently new "spark" pixels appear at the bottom of the matrix. A higher value means more frequent ignition of flame points.
* `const uint8_t turbulence = SEGMENT.custom2;` - Stores the user-defined custom2 value to a variable called turbulence.  This is used to introduce randomness in spark generation or flow — more turbulence means more chaotic behavior.

Next we will look at some lines of code that handle memory allocation and effect initialization:

* `unsigned dataSize = SEGMENT.length(); // allocate persistent data for heat value for each pixel` - This part calculates how much memory we need to represent per-pixel state.  SEGMENT.length() returns the total number of LEDs in the current segment (i.e., cols * rows in a matrix).  This fire effect models heat values per pixel (not just colors), so we need persistent storage — one uint8_t per pixel — for the entire effect.


* `if (!SEGENV.allocateData(dataSize))`
  `return mode_static(); // allocation failed` - This section allocates a persistent data buffer tied to      the segment environment (SEGENV.data).  The syntax SEGENV.allocateData(n) requests a buffer of size n       bytes (1 byte per pixel here).  If allocation fails (e.g., out of memory), it returns false.
  If data allocation fails, the effect can’t proceed.  It calls mode_static() — a fallback effect defined     earlier in the file that just fills the segment with a static color.  We need to do this because WLED       needs a fail-safe behavior if a custom effect can't run properly due to memory constraints.


## Compiling
TODO


## Change Log


