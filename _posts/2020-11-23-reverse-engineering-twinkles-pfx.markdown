---
layout: post
title:  "Reverse Engineering Twinkles PFX"
date:   2020-11-23 11:59:17 -0600
categories: development c++
---

N3V Games's Trainz Railroad Simulator 2019 is a fairly modern game, and yet, it, and all previous iterations in the Trainz series, have roots dating back to Trainz 1.0, from 2000-2001.
These roots range from low level leftovers from 20 years ago all the way to its battle-tested content management tools and its community full of Trainz veterans.
One of the most interesting aspects to me, however, has been Trainz's continued backwards compatibility with legacy formats.

Among these formats is .tfx, a particle system file created by Twinkles - an obscure program made by Kazys Stepanas in 2002.
I believe the program was made with support for Auran's "Jet" engine in mind, but I'm not entirely sure why N3V (then Auran) chose it over JET's built in particle functionality; glancing at the engine documentation, the built in particles seem to be a lot more capable (trails, mesh emitters, etc).
Regardless, it was still a fun challenge to reverse engineer the format and create a viewer/editor program (with hopefully better camera controls than the original Auran tool).

Before starting on the format serialization code or looking into the minutiae of particle rendering optimizations, I decided to first get to know the format and see how it's structured.
This process consisted of a few hours of meticulously exporting and diffing pairs of .tfx files - one having a target value set to its default, and the other with the value set to an easily recognizable integer or float.
After poring over the hex comparisons, I had a pretty comprehensive notepad document that I could base my viewer/editor off of.

The format is structured as following:
{% highlight plaintext %}
XFPT type magic
uint32 Version
uint32 NumEmitters
    XFTS header
    Texture KUID (int32 UserID, uint32 ContentID, uint32 Revision)
    Vec3 Position
    Quat Rotation (xyzw)
    Track<Vector3> Emitter Size (uint32 NumFrames, Frames[float time, Vector3 Value])
    Track<float> Emission Rate
    Track<Vector3> Velocity Cone
    Track<float> Z Speed Variance
    ParticleType Type (FaceCamera 0x1, FaceMotion 0x2, FaceDown 0x4, FaceHorizontal 0x10)
    float unknown
    Track<float> Lifetime
    Track<float> Lifetime Variance
    Track<Vector2> Min/Max Size
    Track<float> Size Variance
    Track<float> Size
    Track<Color> Color (uint8 B, G, R, A)
    Track<float> Max Rotation
    Track<float> Gravity
    float Min Velocity Speed
    float Max Velocity Speed
    Track<float> Wind Factor
    if(Version > 104)
        Vector3 Velocity Dampening
    XFDE footer
{% endhighlight %}

Once that was out of the way, it was time to start working on a C++ application.
To handle reading and writing files, I drew heavy inspiration from Unreal 4's FArchive class.
Similar to FArchive, my archive class overloads the left shift operator for both reading **and** writing.
The archive also contains a private flag that keeps track of whether it's importing or exporting.
These two things allow me to use a single serialization function for all of the particle relevant classes, rather than two separate functions for export and import.
Despite it being initially confusing, this design pattern is actually great for iteration, especially when doing format research, because changing a value's storage behavior won't require careful modifications to both export and import functions (prevents bugs, too)!
I also used a template class to store the keyframe tracks, which allowed me to consolidate any code that retrieved the lower/upper bound frames and interpolated a value between them, despite the difference in types.

After I was more or less confident that the data was reading correctly, I started working on a renderer and interface.
I moved over some old SDL2 boilerplate from one of my previous OpenGL projects, and went with Dear ImGui for The Editor&trade;.
[Learn OpenGL][learnopengl] was, as always, an indispensable resource in quickly looking up the easily forgettable syntax and ordering of GL calls.

<figcaption>Early rendering tests using GL_POINTS</figcaption>
<img src="https://i.ibb.co/6B8N9Bq/ezgif-com-gif-maker-8.gif">


After a few matrix and quaternion headaches, I think I came up with a pretty neat way of rendering the particles without breaking the metaphorical performance bank.
Rather than processing each particle quad on the CPU, I decided to create the particle geometry through OpenGL entirely, taking advantage of graphics hardware.
Instead of sending oriented triangle data to the GPU each frame or using the dreaded fixed function pipeline (*the horror*), I send the data initially to the GPU as a batch of particles (disguised as GL_POINTS vertices), each with a layout that contains all of the necessary details of the particle's type (world position, rotation, size, etc).
When the GPU receives these particles, they're then sent to a really cool geometry shader, which, using the view and projection matrices and some fancy cross product math, transforms the single point into a triangle strip primitive that can face the camera, face the velocity, face down, or billboard (you may notice a similarity here to the tfx particle type enum :wink:).
All of this has the effect of a crazy performance boost, and it only costs one draw call per emitter! (I'm super proud)

<figcaption>Efficient camera-facing quads!</figcaption>
<img src="https://i.ibb.co/qDV2r04/ezgif-com-gif-maker-9.gif">

<figcaption>WIP particle time graph editing</figcaption>
<img src="https://i.ibb.co/M87qnPW/ezgif-com-gif-maker-10.gif">


<img src="https://i.ibb.co/L0fcz7q/ezgif-com-gif-maker-11.gif">

[learnopengl]: https://learnopengl.com/