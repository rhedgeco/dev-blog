---
layout: post
title: (Part 2) Runtime Audio Generation in the Unity Engine - Creating Simple Sounds
thumb: thumbnail.jpg
---

<h1>Table of Contents</h1>

- [🔗 **Introduction**](/2022/unity-audio-generation-fundamentals/#introduction)
- [🔗 **(Part 0) Fundamentals**](/2022/unity-audio-generation-fundamentals/#fundamentals)
- [**(Part 1) Creating Simple Sounds**](#creating-simple-sounds)
  - [Project Setup](#project-setup)
  - [Hooking into the sound engine](#hooking-into-the-sound-engine)
  - [Establishing an architecture](#establishing-an-architecture)
    - [Native Buffers](#native-buffers)

# Creating Simple Sounds
In this post we will cover getting a project set up and start laying down the groundwork for generating sound in the Unity Engine. The [previous post](/2022/unity-audio-generation-fundamentals) covered a basic understanding on what digital sound is, and also provided some insight into the vocabulary we will be referencing. If you have never dealt with digital sound manipulation before, it may be worth your time to go back and take a look.

## Project Setup
Create a new Unity project. The editor used in this post is version **2020.3.19f1**.

Unity comes with a BUNCH of preinstalled packages in the [package manager](https://docs.unity3d.com/Packages/com.unity.package-manager-ui@1.8/manual/index.html). For the sake of keeping things clean, we will remove pretty much all of them, and only add ones we know we will need right now.

![Package manager example](package-manager.png)

One big thing that we need to install is the [**Unity Burst Compiler**](https://docs.unity3d.com/Packages/com.unity.burst@0.2-preview.20/). The burst compiler is some voodoo magic that takes standard C# IL/.NET bytecode and compiles it down to *REALLY* fast native code. We will be using this heavily to generate sounds because performance is *critical* when generating sounds at runtime.

The only other package I have is one for my specific <abbr data-title="Integrated Development Environment">IDE</abbr>, which happens to be [JetBrains Rider](https://www.jetbrains.com/rider/). You dont *need* to use this IDE, but it's my IDE of choice.

With our editor up and running, lets start hooking into the sound engine.

## Hooking into the sound engine
Unity's MonoBehaviour allows for a method called `OnAudioFilterRead`. This is usually used to process incoming audio, but nothing is stopping us from *creating* audio in the first place.

Let's create a new MonoBehaviour and call it `SynthOut`, and add the filter method. This takes in a `float` array and an `int` to represent the number of channels.

We will create chains of sound generators and filters, but this class will ultimately be the point where the sound gets inserted back into Unity's built-in sound engine.

```csharp
public class SynthOut : MonoBehaviour
{
    private void OnAudioFilterRead(float[] data, int channels)
    {
        // sound generation logic goes here
    }
}
```

The `float` array is what we call an audio buffer. This contains the samples for the next section of audio playing in Unity's sound engine. This should currently be set to have **2048** samples. This is because, by default, unity's buffer length is 1024, and there are 2 audio channels *(one for left and right)*.

The order of the data when in 2 channel mode is an interleaving of the samples as a left right repeating pattern *(other modes follow a similar paradigm just with more channels)*:

![interleaving channels](interleaving.jpg)

Now that we have an entry point into the sound engine, and understand the data layout, let's build a system around it to make it easy to expand into many types of sounds.

## Establishing an architecture

### Native Buffers
Since we are using the burst compiler, we cannot be passing around managed objects into Burst Compiled methods. Unfortunately this includes arrays like `float[]`.

We know we need performance and will have to use something other than managed arrays. You also may know of a struct called [`NativeArray`](https://docs.unity3d.com/ScriptReference/Unity.Collections.NativeArray_1.html), but this is unfortunately only able to be used in jobs and *not* burst compiled methods because it has a [`DisposeSentinel`](https://docs.unity3d.com/ScriptReference/Unity.Collections.LowLevel.Unsafe.DisposeSentinel.html) in it. So lets make our own.

We will need a first struct that allocates native memory for us. Let's call it `BufferHandler`. This can be used in many places, so we will keep it generic for now. One downside of not having the DisposeSentinel, is that if we aren't careful about disposing this ourselves, it can leave the data on the heap and start leaking memory. We will need to make sure to keep that in mind when working with this.

```csharp
[StructLayout(LayoutKind.Sequential)]
public unsafe struct BufferHandler<T> : IDisposable where T : unmanaged
{
    public int RawLength { get; }
    public T* Pointer { get; private set; }
    public bool Allocated => (IntPtr) Pointer != IntPtr.Zero;

    public BufferHandler(int rawLength)
    {
        RawLength = rawLength;
        Pointer = (T*) UnsafeUtility.Malloc(RawLength * sizeof(T), 
            UnsafeUtility.AlignOf<T>(), Allocator.Persistent);
    }

    public void Dispose()
    {
        if (!Allocated) return;
        UnsafeUtility.Free(Pointer, Allocator.Persistent);
        Pointer = (T*) IntPtr.Zero;
    }
}
```

Now that we are working in the native world. We will need a way to copy that data back to the managed world efficiently at some point. So let's add a method to our `BufferHandler` to copy data into a managed array. We will have to **pin** the managed array when we copy the memory so that it can't move while we copy. It can be released immediately after.

```csharp
public void CopyTo(T[] managedArray)
{
    if (!Allocated) throw new ObjectDisposedException("Cannot copy. Buffer has been disposed");
    int length = Math.Min(managedArray.Length, BufferLength);
    GCHandle gcHandle = GCHandle.Alloc(managedArray, GCHandleType.Pinned);
    UnsafeUtility.MemCpy((void*) gcHandle.AddrOfPinnedObject(), Pointer, length * sizeof(T));
    gcHandle.Free();
}
```

And while we are at it, lets create a method to copy into its fellow buffers for when we need to move data around later.

```csharp
public void CopyTo(BufferHandler<T> buffer)
{
    if (!Allocated) throw new ObjectDisposedException("Cannot copy. Source buffer has been disposed");
    if (!buffer.Allocated) throw new ObjectDisposedException("Cannot copy. Dest buffer has been disposed");
    int length = Math.Min(BufferLength, buffer.BufferLength);
    UnsafeUtility.MemCpy(Pointer, buffer.Pointer, length * sizeof(T));
}
```

Let's now also create a wrapper for this object to contain our audio buffer data. We will call this our `SynthBuffer`. We will store data in here about the properties of the buffer, so we can keep everything we need in one spot.

```csharp
[StructLayout(LayoutKind.Sequential)]
public struct SynthBuffer : IDisposable
{
    public int Channels { get; }
    public int BufferLength { get; }
    public int ChannelLength { get; }
    public BufferHandler<float> Handler { get; }

    public SynthBuffer(int bufferLength, int channels)
    {
        Channels = channels;
        BufferLength = bufferLength;
        ChannelLength = bufferLength / channels;
        Handler = new BufferHandler<float>(BufferLength);
    }

    public void Dispose()
    {
        Handler.Dispose();
    }
}
```