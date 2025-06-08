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

The first line of the code imports the `[wled.h](https://kno.wled.ge/interfaces/json-api/#effect-metadata)` file into this module.  This file is the 

Currently set in the code have the pixels be all black, but can be modified to take on a different behavior.  For example, say you wanted to...
TODO
 

Pre-loaded in this template is an example 2D Effect called "Diffusion Fire", which is the name that would be shown in the UI once the binary is compiled and run on your device.  We can explore the anatomy of this effect below.





## Compiling



## Change Log


