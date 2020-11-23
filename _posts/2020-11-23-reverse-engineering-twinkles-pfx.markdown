---
layout: post
title:  "Reverse Engineering Twinkles PFX"
date:   2020-11-23 11:59:17 -0600
categories: c++ development
---

N3V Games's Trainz Railroad Simulator 2019 is a fairly modern game, and yet, it, and all previous iterations in the series, have roots dating back to Trainz 1.0, from 2000-2001.
These roots range from design choices to content management tools, but the most interesting to me is its backwards compatibility with legacy formats.
Among these formats is .tfx, a particle system file created by an obscure program called Twinkles.
I'm not entirely sure why N3V (then Auran) decided to use this format and JET plugin to handle particles rather than JET's built in particle functionality; from glancing at the engine documentation, the built in particles seem to be a lot more functional.
Regardless, it was still a fun challenge to reverse engineer the file format and subsequently create a viewer and editor program (with hopefully better camera controls than the original Auran tool).

Before starting on the format serialization code or looking into the minutiae of particle rendering optimizations with OpenGL, I decided to first get to know the format and see how it's structured.
This process consisted of a few hours of meticulously exporting pairs of .tfx files - one having the target value set to its default, and the other with an easily recognizable integer or float.
After this, I had a more or less complete notepad document that I could base my viewer/editor off of.

The format is structured as following (pseudocode):
{% highlight plaintext %}
XFPT type magic
uint32 Version
uint32 NumEmitters
    XFTS header
    Texture KUID (int32 UserID, uint32 ContentID, uint32 Revision)
    Vec3 Position
    Quat Rotation (xyzw)
    //tracks along emitter life (time based emitters?)
    Track<Vector3> Emitter Size (uint32 NumFrames, Frames[float time, Vector3 Value])
    Track<float> Emission Rate
    Track<Vector3> Velocity Cone
    Track<float> Z Speed Variance
    ParticleType Type (FaceCamera 0x1, FaceMotion 0x2, FaceDown 0x4, FaceHorizontal 0x10)
    float unknown (emitter life?)
    //tracks along particle life
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

Once that was done, it was time to start working on a C++ application.
To handle reading and writing files, I drew heavy inspiration from Unreal 4's FArchive class.
Similar to FArchive, my archive class overloads the left shift operator for both reading **and** writing.
The archive also has an internal private member that keeps track of whether it's importing or exporting.
These two things allow me to use a single function, Serialize, for all of the particle relevant classes, rather than two (export and import).
It's a godsend for iteration, especially when doing format research, because changing a value's storage behavior won't require modifications to both the export and import functions(prevents bugs, too!).
I used a template class to store the keyframe tracks, which allowed me to consolidate any code that retrieved the lower/upper bound frames and interpolated a value between them.

After I was more or less confident that the data was reading correctly, I began working on a renderer and interface.
I moved over some old SDL2 boilerplate from one of my previous OpenGL projects, and decided to use Dear ImGui for The Editor&trade;.
["Learn OpenGL"][learnopengl] was, as always, an indispensable resource in quickly looking up the proper syntax and ordering for GL calls.
After a few matrix and quaternion headaches, I think I came up with a pretty neat way of rendering the particles without breaking the metaphorical performance bank.
Rather than processing each particle quad on the CPU, I decided to handle particle geometry through OpenGL entirely!
Instead of sending oriented triangle data to the GPU each frame or using the dreaded fixed function pipeline (*the horror*), I send the data initially to the GPU as a list of particles (disguised as vertices), each with a layout that contains all of the details of the particle's type (world position, rotation, size, etc).
When the GPU receives these particles, they're then sent to a really cool geometry shader, which, based on the view and projection matrices, transforms the single point into a triangle strip primitive that can face the camera, face the velocity, face down, or billboard (you may notice a similarity here to the tfx particle type enum :) ).
This has the effect of a crazy performance boost, and it only costs one draw call per emitter! (I'm super proud)

[learnopengl]: https://learnopengl.com/