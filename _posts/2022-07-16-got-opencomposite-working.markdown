---
layout: post
title:  "I got VRChat working with OpenComposite/Monado/Linux! No more yucky SteamVR!"
date:   2022-07-16 00:00:00 +0100
categories: update
---

Hey friends, this is super cool, somehow I put out enough fires that I can play VRChat on Linux! Check it out:

<iframe title="vrchat with opencomposite/monado" src="https://diode.zone/videos/embed/04d23ef5-db0f-4a40-974a-53ab5cc4abf3" allowfullscreen="" sandbox="allow-same-origin allow-scripts allow-popups" width="560" height="315" frameborder="0"></iframe>

Let me explain this from scratch. If you vaguely know what OpenXR, OpenVR, Monado and libsurvive are, you can skip this part.

Let me define some terms:

* "Runtime" - a piece of software that goes in-between AR/VR games/apps and your actual headset. It's responsible for tracking your head, hands/controllers, initializing/tending to your VR headset, and making your VR headset display images that go into your eyeballs. Usually, every frame, apps ask the runtime for where your head/hands are, and then render an image and give this image back to the runtime to show to you.

* OpenVR: An API that Valve developed to do the above.

* OpenXR: A newer API developed by a lot of people to do the above. Most people would argue that it's better than OpenVR. In my limited experience I'd agree that it's better, but it doesn't matter because I have to use it for reasons.

* SteamVR: A runtime that implements OpenVR and OpenXR, and is specifically designed to work nicely with Valve Index, and Vive headsets, and for supporting community plugins.

* Monado: An open-source runtime that only implements OpenXR. I happen to work on it for my job, and know how to debug it much better than SteamVR.

* OpenComposite: a library that I help develop for fun that pretends to be an OpenVR runtime while really just translating its calls to OpenXR.

Basically... on PC, VRChat is OpenVR only. So, usually, it only works with SteamVR. And SteamVR on Linux *sucks*, like, a lot. Last I tested it it had like 150ms head tracking latency, which was so bad that it didn't even make me sick; it just completely stopped being immersive.

I use Linux for my job, and it's annoying to have to reboot to Windows for this one thing and not have all my regular stuff. Since VRChat purportedly only works with SteamVR and SteamVR for Linux isn't an option, what can we do?

Well there's this thing called [OpenComposite](https://gitlab.com/znixian/OpenOVR). It's mostly developed by [this guy](https://znix.xyz/) but also by a bunch of other people [here](https://discord.gg/2EsWQk8ktp). It's a really cool idea, and since Monado is so familiar to me, would alleviate my problems, if it worked. ZNix recently got the input bindings working well enough that VRChat *worked* with OpenComposite, but there were two big problems:

* VRChat only ran at like 15fps, and indeed *any* Vulkan OpenVR app, even native ones, ran really slow.
* libsurvive, the open-source reverse-engineered implementation of Lighthouse tracking that I use in my setup, for some reason was hogging my Index's microphone device, so I couldn't actually use it to talk.

The second one was an easy & fun fix, and you can see it in all its glory [here.](https://github.com/cntools/libsurvive/pull/274/commits/0ad93398b7ae05922ef9388b47e50fb2f7a49b9d) The interesting thing here is that one of the USB devices that represented the interface for talking to one of the controllers was also somehow a USB microphone device. Meaning that when we tried to grab the talking-to-controllers devices, we also grabbed the microphone device. My PR just goes through libusb's method of seeing "hey what interfaces does this device have" and makes sure libsurvive leaves any audio devices alone. Works great so far.

The first one was very interesting, and caused me to learn a lot more Vulkan. I’m still somewhat ambivalent about whether or not I want to live the life of a graphics programmer, but it’s getting somewhat hard to avoid doing it from time to time.

Anyway, the reason things are hard graphics-wise for OpenComposite relates to 📢📢📢Allocation!!!!📢📢📢 Basically, when the app wants to give an image to the runtime to display to the user, we have to have a spot on the GPU for that image to exist. We have to do weird stuff with graphics APIs to get the image to show up in both the runtime and application process - normally, for security, programs can't see each others' memory, and because of that it's always kind of complicated for two programs to share the same resource at all. With OpenVR, the app creates this image on the GPU (with all the weird flags that specify that Vulkan needs to set it up such that it'll work if we share this image with another process), then sending this image to the OpenVR runtime, once per view (so, usually, twice.) Then, the runtime takes this image, distorts it and displays it on the headset.

So it's not obvious, but it's *really bad* that the app is the one that allocates the swapchain image here. The runtime might want specific flags set on the image so that it can do specific things with it when distorting/doing whatever else is required to the image, but the image has already been allocated, it can't do those now. So, if you have a problem, your best bet is to just copy the image over to a new image you've allocated yourself, as the runtime.

OpenXR does it the opposite way, and has opposite problems - instead, the *runtime* allocates the swapchain images and sends those to the *app*. This might be bad for the app, since maybe the app needs to set specific flags on the image to do specific render passes (I don't know this; don't know enough about gamedev.) In practice this seems to not be a huge problem, but the point is that both ways are annoying, and I refuse to pick a side here.

And they're mutually incompatible with each-other. If we bridge OpenVR and OpenXR, the app allocates an image and writes to it, and the runtime allocates an image and reads from it. These two images are not the same image, so unless we do something the runtime is going to get a black/garbage screen! There are two potential fixes: one, add a crazy layer to Vulkan that checks every `vkCreateImage`, looks for ones that look like they are swapchain images, and instead of allocating something new it just returns the memory backing the runtime's images. This would be hard, so we don't do that. Instead OpenComposite just gets the app's image and the runtime's image, and copies the data from the app to the runtime. On reasonably powerful graphics cards (which we can somewhat expect to have, considering OpenComposite is made by and for PCVR enthusiasts) this performs fine, but it's not ideal.

But wait, it gets more complicated! Not only does the app allocate its own image, but, under Vulkan, it also decides which VkInstance, VkPhysicalDevice, queue, and VkDevice to use. The struct it uses to do that looks like this:

```cpp
struct VRVulkanTextureData_t
{
	uint64_t m_nImage; // VkImage
	VkDevice_T *m_pDevice;
	VkPhysicalDevice_T *m_pPhysicalDevice;
	VkInstance_T *m_pInstance;
	VkQueue_T *m_pQueue;
	uint32_t m_nQueueFamilyIndex;
	uint32_t m_nWidth, m_nHeight, m_nFormat, m_nSampleCount;
};
```

Looks a little bit boring, but there are some horrifying things in here. For one, it sends us the VkDevice, VkPhysicalDevice, VkInstance, VkQueue and queue family index with *every texture*, meaning it can switch those up on us whenever it wants. Meaning that, if you want to be truly conformant, you'd need a bunch of annoying state tracking code to check that it *hasn't* switched those up on us, and if it has, delete all your old swapchain stuff and recreate it with the new handles. Also, let's look at the way OpenXR's Vulkan graphics binding works:

```
typedef struct XrGraphicsBindingVulkanKHR {
    XrStructureType             type;
    const void* XR_MAY_ALIAS    next;
    VkInstance                  instance;
    VkPhysicalDevice            physicalDevice;
    VkDevice                    device;
    uint32_t                    queueFamilyIndex;
    uint32_t                    queueIndex;
} XrGraphicsBindingVulkanKHR;
```

Notice something? 