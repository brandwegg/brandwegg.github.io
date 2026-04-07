---
title: "Recreating the tentacles from returnal"
summary: "For my specialisation at TGE, I decided to recreate an iconic VFX from returnal."
categories: ["Post","Blog",]
tags: ["post"]
#externalUrl: ""
#showSummary: true
date: 2026-03-31
draft: false
---

## Intro
For this project I wanted to recreate an effect I’d seen from Returnal, namely the tentacles they use generously throughout the game, both as environment and parts of enemies.

Without any prior knowledge of generating meshes on the GPU, I delved right in and based my experimentation on the fantastic GDC-presentation by Risto Jankkila and Sharman Jagadeesan from 2022.

The basic concept is to simulate a chain of particles and generate a mesh around it.

## Breakdown

### Setup
Fundamentally, there are 4 buffers in play. Two for the particles, one for the vertices and one for the indices. 
The particles are updated on the GPU and the vertices generated in the same step. 
The reason for two particle buffers is that one is read-only representing the previous frame’s data in order to avoid syncing millions of reads from the same buffer that is being written to. 
As mentioned in the GDC talk, this introduces a lag between each particle and the parent, which can be used as smoothing over time. 
It does come with complications, however, especially with fast movement, long dependency chains and variable frame rates. More on that later.

### Movement
The particles follow a combination of basic behaviours that are applied to the particles' velocities with adjustable weights. These behaviours are:
* Wriggle: a simple sin-wave to generate tentacle-esque movement.
* Straighten: a uniform force along the tentacle according to the orientation of the base.
* Stiffen: a force according to the orientation of the parent in order to reduce the amount of bend at each joint.
* Gravity: a uniform force downwards
* Look-at-camera: a uniform force towards the camera

The goal with these behaviours was mainly to demonstrate the flexibility that comes with blending behaviours and the capability of adjusting them both interactively and programmatically.

{{< video
src="EditorEdit.mov"
loop=true
muted=true
autoplay=true
playsinline=true
>}}

### Mesh

Now that the positioning of the particles is out of the way, we can move on to the orientation. 
Each particle is logically oriented somewhere “in-between” the parent and the child, but how do we find that orientation? 
What I decided to be the “forward” in this situation, the direction along the curve, is straightforward enough: the direction from the parent of the node to the child. 
The “right” direction is trickier since it requires an optimal rotational alignment along the length of the shape to avoid twisting the vertices and thus deforming the triangles.

```hlsl
float3 normal = child.position - parent.position;
float3 n = normalize(normal);
float3 right = 
    normalize(parent.right - n * (dot(parent.right, n) / dot(n,n)));
```

If we rotate a vector around the forward vector we get a ring of points around each particle, or “joint.” This serves perfectly as positions for the vertices. 
Using the right vector as the starting point, we ensure that all the points will be aligned.

```hlsl
void submit_ring_v(uint particleId, float r, float scale, float percentage)
{
    const Particle particle = rwPreviousParticleBuffer[particleId];
   
    const uint startVertex = particleId * verticesPerRing;
    const float3 offset = particle.right * r;
    const float vertexAngle = 2 * TAU / info.triangles_per_segment;
    
    float ringRadius = r * max(scale, FLT_EPSILON);
    const float invRadius = 1.f / ringRadius;
    const float invCircumference = 1.f / (r * TAU);

    [unroll]
    for(uint i = 0; i < verticesPerRing; ++i)
    {
        const float angleNext = Timings.x + vertexAngle*(i + 1);
        const float4 rotator = create_angle_axis(particle.normal.xyz, angleNext);
        const float3 rotated = rotate_vector(offset, rotator);
       
        vertex v;
        v.position = (rotated + particle.position);
        v.normal = (v.position - particle.position) * invRadius;
        v.binormal = particle.right;
        v.uv = float2(float(i)/verticesPerRing, percentage * invCircumference);
        v.padding = 0;

        rwVertexBuffer[startVertex+i] = v;
    }
}

```

For the indices, it helps visualising the vertices.
For every ring, we have 5 vertices. For every ring, we offset the count by 5*n, where n is the index of the current particle. 
Now all we have to do is construct triangles and find a pattern in the data.

```hlsl
// verts of the first particle
0 1 2 3 4
// verts of the second particle
5 6 7 8 9
...
```


This is one way to construct the quads, with permutations. 
I settled on this because it starts off at 0 and has a nice symmetry. 
Note that the two middle columns are mirrored.
```hlsl
// Triangle 1 | Triangle 2
     015          516
     126          627
     237          738
     348          849
     409          905
```

Looking at the left column, the indices are simply counting up from 0. 
The next column and its mirror are simply one higher than that. 
The middle columns are offset by a whole circle of vertices, and the last column one more than that.

```hlsl
void submit_ring_i(uint particleId)
{
    particleId -= 1;
    const uint segmentId = (particleId - particleId / info.particles_per_emitter);
    const uint startIndex = info.vertices_per_segment * segmentId;
    const uint startVertex = particleId * verticesPerRing;


    [unroll]
    for(uint i = 0; i < verticesPerRing; ++i)
    {
        // for every vertex, there is 2 triangles
        const uint offset = startIndex + i * 6;  
        rwIndexBuffer[offset + 0] = startVertex + ( 0 + i) % verticesPerRing;
        rwIndexBuffer[offset + 1] = startVertex + ( 1 + i) % verticesPerRing;
        rwIndexBuffer[offset + 2] = startVertex + ( 0 + i) % verticesPerRing + verticesPerRing;
        rwIndexBuffer[offset + 3] = startVertex + ( 0 + i) % verticesPerRing + verticesPerRing;
        rwIndexBuffer[offset + 4] = startVertex + ( 1 + i) % verticesPerRing;
        rwIndexBuffer[offset + 5] = startVertex + ( 1 + i) % verticesPerRing + verticesPerRing;
    }
}
```

## Following entities
I upload delta position and rotation for each group of tentacles every frame, then translate and rotate the base of each tentacle in the shader.

For demonstration purposes, I made a simple system to showcase the runtime capabilities. 
It changes some parameters for all the tentacles based on a global state. 
I made 3 states: idling, preparing, and jumping. In idle, the tentacles pulsate and waver, 
in preparing they pulsate faster and brighter as well as straightening out, and in jumping they calm down.

## Final product
{{< video
src="JumpingBalls.mov"
loop=true
muted=true
autoplay=true
playsinline=true
>}}
