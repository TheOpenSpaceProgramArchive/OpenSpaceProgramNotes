
![OSP Logo](https://i.imgur.com/BeWsiYp.png)

# OSP Technical Document


Contributors to this document: @Capital-Asterisk

TODO: need some diagrams, review, and a more writing

# Table of Contents

- [Introduction](#introduction)
- [Overview](#overview)
  * [Classes (Brief)](#classes--brief-)
  * [Universe](#universe)
    + [Coordinates](#coordinates)
    + [Active Physics Areas](#active-physics-areas)
    + [Trajectory](#trajectory)
  * [Crafts](#crafts)
    + [Machines](#machines)
    + [Wiring](#wiring)
  * [Units](#units)
- [Resources and Configs](#resources-and-configs)
  * [glTF sturdy](#gltf-sturdy)
  * [Config Files](#config-files)
- [Questions](#questions)
- [Ideas](#ideas)


# Introduction

OpenSpaceProgram (OSP) is an open source initiative to make a spaceflight simulation game similar to KerbalSpaceProgram.


~~It's not rocket science, and a working prototype shouldn't take more than a year to complete.~~


Creating such a project isn't an easy task, and requires careful planning and teamwork.


This document describes class names, data structures, algorithms, and considerations on how to make an implementation of OSP regardless of game engine. There will not be anything on gameplay or art direction. This document attempts to take in account all the ideas listed in previous OSP feature documents, discussions on the Discord channel, and tests from existing repositories.

# Overview

OSP is a simulation of a small universe the player can interact with it through spacecraft of their own design. OSP should be able to simulate orbital mechanics in an explorable solar systems, complete with a star, planets, moons, asteroids, and any astronomical body imaginable. The player will use a variety of parts and their knowledge of physics to build their craft for space missions and expand infastructure. 


**Solution**


At it's core, OSP needs to simulate objects in space and their interactions with each other. Every object in space will be known as a `Satellite`, including all astronomical bodies, spacecraft, or even just virtual reference frames. The movement of Satellites are determined by their `Trajectory`. When a player controls a craft, the area around it has to be loaded into the game engine with physics enabled, handled by the `ActiveArea`.

How the craft itself is represented depends on the game engine, but in general, is a collection of `Part`s. Functionality of these parts come from their `Machine` components. For example, adding a "rocket machine component" to a paperweight part makes it into a functioning rocket. There is also a wiring system that allows parts to control each other.


**Implications**

* OSP can also be used as a base for other space-related games.


## Classes (Brief)


* **Satellite**

   Base class for an object in the universe. Has position, mass, a `Trajectory`, a parent `Satellite`, and children `Satellite`s.

   * **AstronomicalBody (Inherits Satellite)**

      Describes a planet, moon, or star and has many tweakable properties

   * **ActiveArea (Inherits Satellite)**

      Contain the game scene with physics enabled, and loads/unloads `Satellite`s around it into their game engine representations.

   * **Vehicle (Inherits Satellite)**
   
      Wraps game engine objects into a `Satellite`, including the player's Craft and its arrangement of `Part`s.
      
   * **ReferenceFrame (Inherits Satellite)**
   
      Virtual point in space used for barycenters, markers, and various other purposes.

   *List more ideas here*

* **Trajectory**

   A strategy that describes how a `Satellite` moves through space as a function of time. Not associated with the game engine's physics.

   * **Landed (Inherits Trajectory)**
      
      Keeps a `Satellite` locked to its parent's surface.
   
   * **Kepler (Inherits Trajectory)**
   
      Makes a `Satellite` orbit its parent according to Kepler's laws.

   * **NBody (Inherits Trajectory)**
   
      Considers gravity from all physical `Satellite`s and moves accordingly. Not entirely sure how to do this efficiently.

   * **Ballistic (Inherits Trajectory)**
   
      A `Satellite` falling through an atmosphere but isn't loaded in the physics engine.

   *List more ideas here*

* **Part**

   A game engine object, objects, or component that can contain OSP-specific information and `Machine`s.
   

* **Machine**

   Base class for a component of a Part and has `WireInput`s and `WireOutput`s

   * **Rocket (Inherits Machine)**
   
      Applies thrust and drains fuel
   
   * **Decoupler (Inherits Machine)**
   
      Detaches certain attachments when activated through a `WireInput`.
   
   * **Control (Inherits Machine)**
   
      Listens to player inputs and controls other  to `WireOutput`s
      
   *List more ideas here*

* **WireOutput**

   Used by a `Machine` to send data to another `Machine`. Stores a value of some datatype representing throttle percentage, attitude control, etc..., which is read by connected `WireInput`s.

* **WireInput**

   Used by a `Machine` to read data from another `Machine` by connecting to one of its its `WireOutput`.

*List more ideas here*


## Universe

To represent a large universe, it makes sense to use an n-ary tree or hierarchy or Satellites. Satellites can be parented to other Satellites, and all coordinates are relative to their parent. There exists a root satellite at the center of the universe.


Each Satellite also has a Trajectory class which describes how the object moves through space as a function of time, used to implement orbital mechanics. The system is open to different methods of calculating orbits, such as Patched Conics or n-Body. A wandering spacecraft that moves around has its Trajectory swapped out depending on its surroundings. "Landed on a surface" is also a Trajectory class.


The player interacts with the universe through the Game Engine. This requires a special Satellite known as an `ActiveArea`. ActiveAreas contain the game scene with physics enabled, and loads/unloads Satellites around it into their game engine representations.


The large universe consisting of Satellites is referred to as the "OSP universe" in this document.


### Coordinates

In the real world, our solar system is 4.5 billion kilometers in radius,

Using floating points, Precision will decrease as distance from the origin increases. At ±16,777,216 units, 32-bit floats will no longer be able to represent odd integers. Floats work best when they're close to zero, which is wasted as zero is the insides of a planet.

**Solution**

Why not make a coordinate system that can scale down to the quantum level, but also be able to span lightyears? Well this is a working solution, I guess.

Again, coordinates of a Satellite are relative to its parent. Coordinates are in signed 64 bit ints. How much units is equal to a meter, is defined by the parent Satellite's "precision" value, discussed later. Note that this is separate from the game physics and scene system, which likely uses floats.

In general,

* Universe is a tree of Satellites
* Signed 64-bit ints used as XYZ coordinates
* All coordinates are relative to parent
* Coordinates can vary in precision

Satellites control the precision of their children's coordinates, or how many 64-bit int increments is equal to a meter. The precision value is an exponent to 2. Increasing precision decreases maximum space around the Satellite, decreasing precision increases this maximum space. Precision of coordinates will always be a power of two, because that's what the robots like. Precision should be set to a high value for small planets with moons and landed crafts as children, and a low value for larger objects like stars with orbiting planets. The values are absolute and do not accumulate along ancestors.

| precision   | Meters per unit    |
|-------------|--------------------|
| -3          | 8                  |
| -2          | 4                  |
| -1          | 2                  |
| 0           | 1                  |
| 1           | 1/2                |
| 2           | 1/4                |
| 3           | 1/8                |
| 4           | 1/16               |
| .. 10       | 1/1024             |


Implications:

* Relative coordinates between Satellites can be calculated through finding a common ancestor.
* Satellites like the player's Vehicle, can change parents to nearby objects, switching to a different coordinate system, which works nicely for patched conics.

Again, if you don't like this implementation then go to yell at @Capital-Asterisk.

### Active Physics Areas

The `ActiveArea` is a special kind of Satellite which stores the physics-enabled game scene, projecting the OSP universe into a tiny space the game engine can handle. The player can interact with the game scene through controlling Vehicles, and changes will be made to the OSP universe accordingly. ActiveArea is positioned the same way as any other Satellite, but usually follows the player around.

**Loading and Unloading Satellites**

All Satellites have some "load radius" variable, describing a sphere around the satellite. When the sphere of a normal Satellite hits an ActiveArea's sphere, then the ActiveArea calls the Satellite's `load()` functions, requesting that it should load a representation of itself into the game scene. Whatever happens in `load()` is up to the Satellite's implementation. For example, a Planet should load terrain into the game scene when an ActiveArea calls its `load()`.

Note: maybe make the sphere a cube instead

**Loading Previews**

The ActiveArea should also load previews of distant objects, such as rendering planets as normal spheres, and spacecraft as a bright light flare.

**Floating Origin**

The (0, 0, 0) position of the ActiveArea's internal game scene, corresponds to the center of the Satellite in the OSP universe.
ActiveArea keeps units of its Scene in meters, regardless of any precision values.

**Implications**

* There can be multiple active areas, aside from supporting distant objects having active physics, it is essential for multiplayer.

### Trajectory

A Trajectory class describes the movement of Satellites when they're not loaded in an ActiveArea. Satellites should be able to follow many different kinds of paths in space. They can be can be orbiting, landed, falling through an atmosphere, etc... Hard-coding orbital mechanics into the Satellite just isn't right. Trajectories are interchangable strategies that make it easy to manage the state of a Satellite, as well as allow for different kinds of orbit solvers, such as N-body, to be implemented cleanly.

**Some ideas for a Trajectory:**
* Landed Trajectory: for Satellites landed on a surface
* Kepler Orbit Trajectory: Used for Patched Conics
* N-body Trajectory: RK4 intergrators or something


**Some crazy ideas:**
 * In-Flight Autopilot Trajectory: Plane eats fuel while moving through the atmosphere at constant altitude
 * Parabolic Flat Earth Trajectory: because why not ¯\\_(ツ)_/¯

**Implications**

* Some trajectories don't have to be updated every frame, and can just be functions of elapsed time or something.


For an example of how this system might work, let's say a player is landed on the moon. The craft has active physics and is loaded, but is stationary and not moving. The player exits to the menu, causing the lander to get unloaded. The lander's Trajectory is then set to a LandedTrajectory, which only saves its spot on the surface. The player comes back to the lander after a few minutes, and the moon had rotated pretty far. For the game to know the lander's new position, it calls its Trajectory, which is set to a LandedTrajectory. LandedTrajectory calculates the craft new position based on it's stored relative position, and the current rotation of the planet.

When the player takes off from the moon, the ActiveArea handles the physics up to orbit. The player exits to the menu again, unloading the craft. The craft is then automatically assigned a KeplerOrbitTrajectory set by configs.


## Crafts

TODO

### Machines

TODO

### Wiring

Parts have to respond to inputs. For example, rockets need a throttle and gimbal inputs. It may make sense to hard-code user input into the parts or craft directly, but that solution isn't future-proof, especially when considering addons. Inputs would quickly turn into a mess of if statements. How about parts that control other parts, such as an autopilot? What if the player wants to control the throttle of each rocket individually, or wants them to respond to different buttons? Or what if the player only wants to control a small portion of a craft instead of destroying the entire space station? There has to be a system that connects parts in an organized way.

**Solution**

A modular solution, is some sort of logic/wiring system, where a Part's Machines can be black boxes with inputs and output that can be dynamically connected to other Machines.

```
Familiar Examples (data types and Machines not discussed for simplicity):

Rocket Engine:
- Inputs: [Throttle] [Gimbal] [On/Off] [etc..]
- Outputs: [Temperature] [Current Thrust] [etc..]

Capsule:
- Inputs: [Oxygen valve to kill crew] [I can't think of any] [etc...]
- Outputs: [Throttle control] [Rotation control] [Stage] [Custom button] [# of crew] [etc...]

Autopilot computer
- Inputs: [On/Off]
- Outputs: [Throttle control] [Rotation control] [Oxygen control] [etc...]
```

TODO: table for different kinds of supported datatypes

**Typical use**

A player can control parts that have a "Control Machine," such as a capsule, so that the player's input devices are interfaced on WireOutputs. These outputs can be connected to the throttle and gimbal WireInputs of the rockets.

If the player wanted to connect a rocket to [custom button] instead, then the player can disconnect the throttle control of the rocket, and connect it to custom button instead.

```
Quadcoupler with differential thrust:
The 4 rockets connected to this part will be automaticall wired to custom throttle controls, so that they respond to rotation controls.
- Inputs: [Rotation control]
- Outputs: [Throttle A] [Throttle B] [Throttle C] [Throttle D]

```

**Implications**



## Units

Use SI units and meters, no argument here. 

TODO

# Resources and Configs

TODO

## glTF sturdy

TODO

## Config Files

TOML?

TODO

# Questions

*How would X be handled?*

TODO

# Ideas

TODO

* Port Bulleted List Extender Continued
