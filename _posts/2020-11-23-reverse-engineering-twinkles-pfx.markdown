---
layout: post
title:  "Reverse Engineering Twinkles PFX"
date:   2020-11-23 11:59:17 -0600
categories: c++ development
---

N3V Games's Trainz Railroad Simulator 2019 is a fairly modern game, and yet it, and all previous iterations in the series, have roots dating back to Trainz 1.0, from 2000-2001.
These roots range from design choices to content management tools, but the most interesting to me is its backwards compatibility with legacy formats.
Among these formats is .tfx, an particle system file created by an obscure program called Twinkles.
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