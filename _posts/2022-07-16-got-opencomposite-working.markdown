---
layout: post
title:  "I got VRChat working with OpenComposite/Monado/Linux! No more yucky SteamVR!"
date:   2022-07-16 00:00:00 +0100
categories: update
---

Hey friends, this is super cool, somehow I put out enough fires that I can play
VRChat on Linux without using SteamVR! Check it out:

<iframe title="vrchat with opencomposite/monado"
src="https://diode.zone/videos/embed/04d23ef5-db0f-4a40-974a-53ab5cc4abf3"
allowfullscreen="" sandbox="allow-same-origin allow-scripts allow-popups"
width="560" height="315" frameborder="0"></iframe>

Let me explain this from scratch. If you vaguely know what OpenXR, OpenVR,
Monado and libsurvive are, you can skip this part.

Let me define some terms:

* "Runtime" - a piece of software that goes in-between AR/VR games/apps and your
  actual headset. It's responsible for tracking your head, hands/controllers,
  initializing/tending to your VR headset, and making your VR headset display
  images that go into your eyeballs. Usually, every frame, apps ask the runtime
  for where your head/hands are, and then render an image and give this image
  back to the runtime to show to you.

* OpenVR: An API that Valve developed to do the above.

* OpenXR: A newer API developed as a Khronos specification by many different
  groups, notably including Valve and Collabora, to do the above. Most people
  would argue that it's better than OpenVR. In my limited experience I'd agree
  that it's better, but it doesn't matter because I have to use it for reasons.

* SteamVR: A runtime that implements OpenVR and OpenXR, and is specifically
  designed to work nicely with Valve Index, and Vive headsets, and for
  supporting community plugins.

* Monado: An open-source runtime that only implements OpenXR. I happen to work
  on it for my job, and know how to debug it much better than SteamVR.

* OpenComposite: a library that I help develop for fun that pretends to be an
  OpenVR runtime while really just translating its calls to OpenXR.

Basically... on PC, VRChat is OpenVR only. So, usually, it only works with
SteamVR. And SteamVR on Linux *sucks*, like, a lot. Last I tried it, it had
around 150ms head tracking latency, which was so bad that it didn't even make me
sick; it just completely stopped being immersive.

I use Linux for my job, and it's annoying to have to reboot to Windows for this
one thing and not have all my regular puter stuff that I've got on my Arch
install. Since VRChat purportedly only works with SteamVR and SteamVR for Linux
isn't an option, what can we do?

Well there's this thing called
[OpenComposite](https://gitlab.com/znixian/OpenOVR). It's mostly developed by
[this guy](https://znix.xyz/) but also by a bunch of other people
[here](https://discord.gg/2EsWQk8ktp). It's a really cool idea, and since Monado
is so familiar to me, would alleviate my problems, if it worked. ZNix recently
got the input bindings working well enough that VRChat *worked* with
OpenComposite, but there were two big problems:

* VRChat only ran at ~15fps, and indeed *any* Vulkan OpenVR app, even native
  ones, ran really slow.
* libsurvive, the open-source reverse-engineered implementation of Lighthouse
  tracking that I use in my setup, for some reason was hogging my Index's
  microphone device, so I couldn't actually use it to talk.

The second one was an easy & fun fix, and you can see it in all its glory
[here.](https://github.com/cntools/libsurvive/pull/274/commits/0ad93398b7ae05922ef9388b47e50fb2f7a49b9d)
The interesting thing here is that one of the USB devices that represented the
interface for talking to one of the controllers was also somehow a USB
microphone device. Meaning that when we tried to grab the talking-to-controllers
devices, we also grabbed the microphone device. My PR just goes through libusb's
method of seeing "hey what interfaces does this device have" and makes sure
libsurvive leaves any audio interfaces alone. Works great so far.

The first one was very interesting, and caused me to learn a lot more Vulkan.
I’m still somewhat ambivalent about whether or not I *want* to live the life of
a graphics programmer, but it’s getting somewhat hard to avoid doing it from
time to time.

Anyway, the reason things are hard graphics-wise for OpenComposite relates to
📢📢📢Allocation!!!!📢📢📢 Basically, when the app wants to give an image to the
runtime to display to the user, we have to have a spot in GPU memory where that
image can live. We have to do weird stuff with graphics APIs to get the image to
show up in both the runtime and application process - normally, for security,
programs can't see each others' memory, and because of that it's always kind of
complicated for two programs to share the same resource. With OpenVR, the app
creates this image on the GPU (with all the weird flags that specify that Vulkan
needs to set it up such that it'll work if we share this image with another
process), then sending this image to the OpenVR runtime, once per view (so,
usually, twice.) Then, the runtime takes this image, distorts it and displays it
on the headset.

So it's not obvious, but it's *really bad* that the app is the one that
allocates the swapchain image here. The runtime might want specific flags set on
the image so that it can do specific things with it when distorting/doing
whatever else is required to the image, but the image has already been
allocated, it can't do those now. So, if there's a problem with the
app-allocated image, the runtime pretty much has to just copy the image over to
a new image you've allocated itself. And there doesn't seem to be a way in the
OpenVR spec for the runtime to encourage apps to allocate images "correctly" so
it's a bit of a wild west situation.

OpenXR does it the opposite way, and has opposite problems - instead, the
*runtime* allocates the swapchain images and sends those to the *app*, and the
app has to write rendered images into them. This might be bad for the app, since
maybe the app needs to set specific flags on the image to do specific render
passes (I don't know this; don't know enough details; could be wrong.) In
practice this way seems to have less problems, but I'm not completely confident
about that.

Importantly for us, they're mutually incompatible with each-other. If we bridge
OpenVR and OpenXR, the app allocates an image and writes to it, and the runtime
allocates an image and reads from it. These two images are not the same image,
so unless we do something the runtime is going to get a black/garbage screen!
There are two potential fixes: one, add a crazy layer to Vulkan that checks
every `vkCreateImage`, looks for ones that look like they are swapchain images,
and instead of allocating something new it just returns the memory backing the
runtime's images. This would be hard, so we don't do that. Instead OpenComposite
just gets the app's image and the runtime's image, and copies the data from the
app to the runtime. On reasonably powerful graphics cards (which we can somewhat
expect to have, considering OpenComposite is made by and for PCVR enthusiasts)
this performs fine, but it's not ideal.

But wait, it gets more complicated! Not only does the app allocate its own
image, but, under Vulkan, it also decides which VkInstance, VkPhysicalDevice,
VkQueue, and VkDevice to use. The struct it uses to do that looks like this:

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

Looks a little bit boring, but there are some horrifying things in here. For one, it sends us the `VkDevice`, `VkPhysicalDevice`, `VkInstance`, `VkQueue` and queue family index with *every texture*, meaning it can switch those up on us whenever it wants, including in-between frames. Meaning that, if you want to be truly conformant, you'd need a bunch of annoying state tracking code to check that it *hasn't* switched those up on us, and if it has, delete all your old swapchain stuff and recreate it with the new handles. (And then we have questions about well should we keep the old swapchain at all? is it going to randomly switch back? Very ugh.) Also, let's look at the way OpenXR's Vulkan graphics binding works:

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

Notice something? For some reason unknown to me, OpenVR asks for a VkQueue and a queue family index, instead of just a queue index and queue family index. Which meant that when the original Vulkan compositor implementation was written in OpenComposite, I think they thought that we *had* to use different instances, devices and queues for the app and runtime handles. This is *bad*, because you can't copy straight across devices. (Actually I found that you *can*, at least under RADV, but it's undefined behaviour under the Vulkan spec so a) it doesn't pass validation layers and b) you might be causing all kinds of problems elsewhere.) Instead, the original impelementation allocated a *third* texture with all the flags set such that you can export its memory across VkDevices. This worked, but the second copy + waiting for stuff to synchronize across devices was *slooowww*.

During my initial foray into this, I had no idea about the rules for importing and exporting buffers, and this third image came as a big surprise and source of confusion. *Why the hell was this thing here? How is this not an obvious inefficiency? There's no way there's not a good reason for this.* I initially tried copying straight from the app image to the runtime image, and it *worked*, but when I ran validation layers I discovered the reason:

```
VUID-vkCmdCopyImage-commonparent(ERROR / SPEC): msgNum: -1823964070 - Validation Error: [ VUID-vkCmdCopyImage-commonparent ] Object 0: handle = 0x7f795f70, type = VK_OBJECT_TYPE_INSTANCE; | MessageID = 0x9348845a | Object 0x60fe2400000009a9 of type VkImage was not created, allocated or retrieved from the correct device. The Vulkan spec states: Each of commandBuffer, dstImage, and srcImage must have been created, allocated, or retrieved from the same VkDevice (https://www.khronos.org/registry/vulkan/specs/1.3-extensions/html/vkspec.html#VUID-vkCmdCopyImage-commonparent)
    Objects: 1
        [0] 0x7f795f70, type: 1, name: NULL
```

UH. OH. Yeah there is your problem. Vulkan is so explicit and conservative about what you're allowed to do that you often end up in really sticky situations like this, as compared to, say, OpenGL or D3D10.

Okay, so... shit... what do we do? We can wait for the app to give us an image, initialize our OpenXR session with the app's `VkInstance`, `VkPhysicalDevice` and `VkDevice` (and hope it doesn't switch these up on us (none of the apps I've tested do this FWIW)), but how do we get the right queue? And what if there are other pitfalls?

After many, many, many false starts, and many hours discussing with community members and coworkers (especially Ryan Bjorn and Christoph, thank you guys so so so so much for being so patient with me and taking the time to really think about my problems, no way I could have done this without your help & advice), I realized you can do this:

```cpp
static void find_queue_family_and_queue_idx(VkDevice dev, VkPhysicalDevice pdev, VkQueue desired_queue, uint32_t& out_queueFamilyIndex, uint32_t& out_queueIndex)
{
	uint32_t queue_family_count;
	vkGetPhysicalDeviceQueueFamilyProperties(pdev, &queue_family_count, NULL);

	std::vector<VkQueueFamilyProperties> hi(queue_family_count);
	vkGetPhysicalDeviceQueueFamilyProperties(pdev, &queue_family_count, hi.data());
	OOVR_LOGF("number of queue families is %d", queue_family_count);

	for (int i = 0; i < queue_family_count; i++) {
		OOVR_LOGF("queue family %d has %d queues", i, hi[i].queueCount);
		for (int j = 0; j < hi[i].queueCount; j++) {
			VkQueue tmp;
			vkGetDeviceQueue(dev, i, j, &tmp);
			if (tmp == desired_queue) {
				OOVR_LOGF("Got desired queue: %d %d", i, j);
				out_queueFamilyIndex = i;
				out_queueIndex = j;
				return;
			}
		}
	}
	OOVR_ABORT("Couldn't find the queue family index/queue index of the queue that the OpenVR app gave us!"
	           "This is really odd and really shouldn't ever happen");
}
```

NO!!!!

<html>
      <img src="/assets/images/notmade.jpg" alt="BAD IDEA" height="300">
</html>

So, this code just assumes there's an isomorphism between queues and queue indices, iterates over all the queue family indices and queue indices that in the VkDevice that the app provides us, and compares their handles to the VkQueue that the app gave us. Problem is that I don't know if the Vulkan spec does guarantees that queues/queue indices we ask for and the VkQueue handles that the Vulkan implementation gives out are isomorphic! RADV does seem to have this isomorphism, but do all of them? I guess RADV probably allocates all the handles when you run VkCreateInstance and just keeps handing out the same ones, but does it *have* to? Will this break on other Vulkan implementations? I have no idea!


Aaannnyyywaaayyyy, with that house-of-cards fix in place, I was able to really cleanly delete OpenComposite's vulkan compositor's third-buffer-allocation stuff and just copy directly from the app's image to the runtime's image. It asserts if the app changes up it's VkInstance/VkDevice/VkQueue, but so far no apps do that, and so far no users downstream have had real problems with it. That MR is [here](https://gitlab.com/znixian/OpenOVR/-/merge_requests/35) for all the world to see.

Anyway, after that, VRChat (and the hellovr_vulkan sample) just started working quite well! There are lots of problems sure, but in general the UIs work well (even through *two* layers of emulation - Proton *and* OpenComposite,) my avatar displays correctly, I can talk to people and while there is some jank, there aren't really any showstoppers left, I can just use it.

However the compositor is still trash. It's doing a whole lot more `vkQueueWaitIdle`s than it needs to, which makes the copies take around 3ms in total - too long. When big games put the GPU under pressure and/or take over about 5ms to render (assuming the displays run at 120Hz here,) this can make the frames tank a little more than they would under a good compositor. I can fix this but I need to do some rather gnarly refactors first. - specifically, I really want us to wait on the queue only on the *next* frame, so that a) we can safely do both copies concurrently and b) bundle up the wait on the image copy with the other stuff Monado does internally such that the next-frame-wait only actually happens very rarely. It'll happen, just will take time.

Also, we aren't yet reporting the correct color spaces to Monado. So, when VRChat gives us sRGB images, we still report them as linear and get too-bright-colors. That'll be fixed very soon.

Aand a whole lot more - here's my crazy long todo:
* SPDX Copyright headers
* Try to collect global state into structs and do a bunch of `magic_global_state = new GlobalStateXY()`
* Remove some indirection between compositor, OpenVR wrapper, and OpenXR wrapper
* Add fallback for if an evil app switches up the VkInstance/VkPhysicalDevice/VkDevice/VkQueue between frames
* Upstream Perfetto/Percetto tracing
* Write a bridge between XR_EXT_hand_tracking and OpenVR skeletal input
* Knuckles controller profile support
* cmake_in instead of compiler options
* Continuous integration on GitLab MRs
* fix colour spaces on Vulkan

todo on Monado:
* Playspace mover
* Make aim->grip offset perfectly match SteamVR

I also need to test with Neos, Beat Saber and Population One. I already know Neos doesn't work for non-graphics reasons (something with the input profile?) but the other ones are big unknowns for me. All of this constitutes a few weeks of full-time work, so I'll be getting through it rather slowly along with my real job and other things I do for fun. Anyways that's pretty much all I've got for this post. I'll probably make another post showing off further fixes I make, and hopefully make my own guide showing how to build all the stuff you need for this. laters!

