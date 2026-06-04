---
id: 1
title: "Inside the Linux DRM Subsystem"
subtitle: "Why DRM replaced fbdev and how the new display pipeline works"
date: "2026.06.03"
tags: "DRM, Linux Graphics"
---

The display peripheral is one of the most important part of a computer. As such,
the display technology has undergone great evolution in capability and
complexity over the years. Of course in the Linux kernel,
this fact holds true as well.

## Brief history of display stack in linux

The very first way Linux officially supported graphics was through the [fbdev](https://en.wikipedia.org/wiki/Linux_framebuffer)
(framebuffer device) framework. The legacy fbdev framework uses a specific region
in memory reserved as a framebuffer(fb). User programme first write data to a fb
via graphics library like [Vulkan](https://en.wikipedia.org/wiki/Vulkan). The fb driver will then read the pixel data,
send it to specific chip that control the display (also known as the display controller)
in the format that the controller expect. This process includes writing specific values
to specific registers to wake it up, set the active window, etc. Although simple,
fbdev is still used today for simple display, such as a small
TFT LCD display commonly used by hobbyist nowadays, connected to a RaspberryPi.

This works great when there's only simple 2D-based applications at the time.
However in the 90s, there was a massive 3D boom in the industry, where the GPU
became more complex and can calculate 3d geometry in hardware.

To accommodate this, the new Direct Rendering Manager (DRM) framework
was created. Originally, it acted as the middleman between the kernel and
GPU for 3D related tasks, where 3D applications and games pass commands
to the GPU via the kernel.

At the time, the mode setting of the display(like its resolution)
was handled by userspace (UMS) by user programme like the [X Server](https://en.wikipedia.org/wiki/X.Org_Server).
This works for some use cases, but there is a huge problem when user wanted to
switch between the GPU-heavy graphics like a game, and simple 2D CPU-based
graphics like the terminal. When switching to a text terminal, the kernel
had to ask the user-space X Server to release control of the GPU.
If a heavy game was running and bogging down the X Server, this communication
would experience massive delays. If the kernel tried to force the text terminal
onto the screen anyway, both programs would write conflicting commands
to the GPU registers simultaneously, instantly locking up the hardware.

In 2008, Kernel Mode Setting ([KMS](https://www.kernel.org/doc/html/v4.15/gpu/drm-kms.html#)) was introduced to fix frequent system crashes
when switching between graphical environments and text consoles.
Moving this logic to the kernel fixes the issue because the kernel operates
with the highest execution privilege. It does not have to wait for permission
from the X Server or for a heavy game to finish a frame. Instead, the kernel
instantly freezes the desktop's graphics state, saves the GPU registers
to kernel memory, and immediately rewrites the hardware registers
to display the text console.

Around the 2010's, the SoC and smartphone boom happened, and it increased
the complexity of displays even more. The KMS now needs to handle
multi-layered overlays for wallpaper, cursor, video playback, etc.
To accommodate this, atomic KMS was introduced in 2015. Through this,
it can now update multiple planes at the same time through a single transaction.
This prevent a plane from being updated while the other is not yet finished,
of which both of them will be rendered at a synchronized timing
when they are both ready by a transaction commit.

## Why DRM instead of the legacy framebuffer

Let's look into more detail of how modern DRM replaces the fbdev framework.
The old approach works for small screen that just have to push pixel to a display.
However, it severely limits modern systems because
multiple applications cannot access the same memory at the same time.

### The Legacy Bottleneck

If a web browser want to display an image under fbdev, it must store the pixel data
of that image in it's own private memory first. The kernel provides one dedicated
memory region for the display. To get the image on screen, the browser must use
the CPU to physically copy its pixel data into that dedicated framebuffer.
If you add a compositor (the software responsible for combining all
individual application windows into the final image you see on your screen)
to this mix, the compositor has to constantly copy data from every
open application into a single master buffer, and then copy that master buffer
to the screen. This wastes processing power and causes screen tearing.

### The GEM Solution

There is a better way to handle memory than constant copying. DRM solves this
using the Graphics Execution Manager (GEM). GEM dynamically allocates
graphics buffers and enables true zero-copy sharing.

Instead of copying pixel data, GEM relies on a concept every programmer knows well: passing by reference.
When an application needs to draw graphics, GEM allocates a chunk of GPU memory for it.
The application does not get raw access to the physical memory addresses.
Instead, the kernel gives the application a secure "handle",
which acts like a specialized pointer.

This architecture becomes incredibly efficient when multiple programs need to work together.
If a web browser wants to hand its finished window to the compositor, it does not copy the pixel data.
Instead, it asks the kernel to package that memory handle into
a standard Unix file descriptor using a mechanism called dmabuf.

The browser sends this file descriptor to the compositor.
The compositor imports it and instantly gains direct access to the exact same
physical memory region. Because everything in Linux is treated as a file,
dmabuf provides a secure, standardized way for completely different processes to look at the same graphics buffer.
No data is duplicated, and no CPU cycles are wasted on copying.

### Compositor

![How compositor manage application buffers to hand it to DRM](/images/drm_gem_compositor.jpeg)

Because of GEM, the modern compositors that uses the [Wayland](https://wayland.freedesktop.org/) protocol
can do its job without the heavy copying required by fbdev.
When an application like your terminal finishes drawing its UI,
it uses GEM to hand a memory reference directly to the compositor.

The compositor collects these buffer references from all open applications and
figures out which window goes where. It then uses the GPU to blend them
into a single master buffer.

Finally, the compositor issues commands to the kernel's KMS subsystem.
It instructs the kernel to map this master buffer to the primary hardware plane for display.
The compositor is also smart enough to keep fast-moving elements separate.
It can tell the kernel to map the mouse graphic directly to a separate physical cursor plane.
This allows the display controller hardware to overlay the cursor seamlessly
without needing to redraw the entire master buffer just because the mouse moved.

## The DRM pipeline

In the DRM system, data flows in a pipeline. It starts from the plane, CRTC, encoder
and finally connector. Each component does different things, but all components must be present
in a drm driver to display the frame successfully. Note that display refers to the physical panel of the screen, while display
controller is the chip that drives the display.

### Plane

A plane contains raw pixel data allocated and written to by userspace in a buffer handled by GEM.
A pipeline can have multiple planes for multiple layers of the UI. There are three types of plane in the kernel: primary, cursor, and overlay.
A pipeline must have at least one plane. For example, a simple display might only need the primary plane,
but something like a desktop environment must also have the cursor plane, so that the GPU hardware can blend (composite) it
with the other planes later in the pipeline without re-rendering the entire frame when only the mouse cursor moves.
The overlay plane is used for video playback or any additional UI layer that the hardware can composite independently.

### Cathode Ray Tube Controller (CRTC)

CRTC is a legacy term from back when computers used CRT monitors. Back then,
images were drawn by an electron gun blasting light onto a phosphor screen,
and the CRTC handled the electrical timing of when to stop blasting light at the end of the row,
and reset the light to the first column of the next row.
Even though modern displays don't work like that anymore, it was kept in the kernel
as an architectural name for the component that drives the electrical timing and sync signals of composited frame
by the display controller toward the encoder.
From this point onward, we are referring to this modern
component as CRTC.

CRTC derives the refresh rate based on the display chip's clock frequency
and the total number of timing intervals per frame, both of which are captured in the display mode.
Display mode refers to the settings that the specific display chip needs
to work with a specific display. Here is a snippet of the drm_display_mode struct:

``` C
struct drm_display_mode {
 int clock;  /*in kHz*/
 u16 hdisplay;
 u16 hsync_start
 u16 hsync_end;
 u16 htotal;
 u16 hskew;
 u16 vdisplay;
 u16 vsync_start;
 u16 vsync_end;
 u16 vtotal;
 u16 vscan;

 int crtc_clock;
 u16 crtc_hdisplay;
 u16 crtc_hblank_start;
 u16 crtc_hblank_end;
 u16 crtc_hsync_start;
 u16 crtc_hsync_end;
 u16 crtc_htotal;
 u16 crtc_hskew;
 u16 crtc_vdisplay;
 u16 crtc_vblank_start;
 u16 crtc_vblank_end;
 u16 crtc_vsync_start;
 u16 crtc_vsync_end;
 u16 crtc_vtotal;
};
```

The struct above contains two versions of each timing variable: those without a prefix represent the plain timings,
which is what userspace and the display specification describe, while those with the crtc_ prefix represent the hardware timings
that the kernel actually programs into the display controller registers. The distinction exists to handle legacy display behaviors
such as interlacing and double-clocking. An interlaced display like one running at 1080i,
common in broadcast television to reduce bandwidth, splits each frame into odd and even lines
and draws them in alternating passes, which requires the kernel to adjust the raw timing values before programming the hardware.

![Display during active, front porch, sync pulse and back porch](/images/drm_crtc_timing.jpeg)

The CRTC drives pixels starting from the top-left, row by row, left to right, until the full frame is drawn.
After each row of active pixels, there is a brief delay called the front porch before the sync pulse.
The front porch exists to give the display electronics time to settle after the last active pixel
before the sync pulse fires, preventing the electrical transition from corrupting the final pixels of the row.
The sync pulse itself is the hardware signal that marks the end of that row.

Following the sync pulse is the back porch, another short delay before pixel output resumes on the next row.
The back porch gives the display time to physically reposition its scanning mechanism
to the start of the next row and stabilize before it has to start receiving active pixel data again.
The same structure applies vertically at the end of each full frame.

Together these periods form the blanking interval, and their durations are what hsync_start,
hsync_end, htotal (and their vertical equivalents) encode.
In order, the sequence is: active pixels, front porch, sync pulse, back porch,
then active pixels again for the next row or frame.

### Encoder

The CRTC outputs a parallel stream of pixels, where each pixel's color channels
and timing signals travel on separate wires simultaneously.
For example, a parallel RGB signal uses 24 data lines (8 bits per channel) plus
additional lines for the clock, hsync, and vsync signals.
This works internally on a chip, but a physical cable like HDMI cannot carry that many wires.

The encoder's job is to take this parallel stream and translate it into
a serial transmission protocol that the connector supports, such as HDMI,
DisplayPort, DSI, or DPI. Most of this translation happens in dedicated
encoder hardware rather than in the DRM encoder struct itself,
since hardware is significantly faster for this kind of work.
The DRM encoder struct simply determine which protocol to use
and passes the signal along to the connector.

### Connector

The connector represents the physical output port of the pipeline,
such as an HDMI or DisplayPort socket. When a display is connected,
the connector reads the EDID, a data structure that contains metadata
about the display such as supported resolutions, refresh rates, and color formats.
This information is exposed to userspace so that it can select an appropriate display mode,
which is then passed back to the CRTC to configure the timing accordingly.
The connector also monitors hotplug events, detecting when a display is connected
or disconnected after boot and notifying the rest of the pipeline so the display mode can be updated.
The drm_connector struct captures all of this, holding the parsed EDID data,
the connection status, hotplug polling mode, and the encoder the connector is currently paired with.

## How Driver Uses the DRM

When writing a display driver, developers do not have to build everything from scratch.
The Linux DRM subsystem provides a massive toolbox of built-in "helpers"
to handle the repetitive boilerplate code. Which helpers a driver actually uses
depends entirely on how complex the physical display hardware is.

For basic displays, like the small LCD screens used by hobbyists and embedded developers,
the kernel offers the drm_simple_kms helpers. Instead of manually configuring every stage of the display pipeline,
a driver can just call drm_simple_display_pipe_init(). This single function automatically wires up the plane,
the CRTC, the encoder, and the connector into one neat package. If the display controller follows standard mobile protocols,
the developer can also plug in mipi_dbi helpers to handle the communication.
This approach allows a basic screen to start drawing pixels with a surprisingly small amount of code.

However, modern graphics hardware is a completely different beast.
A complex display controller like the i915 driver used in most Intel laptops cannot rely on these simple pipes.
Advanced hardware needs absolute control over the entire system to manage strict power saving states,
blend multiple graphical planes simultaneously, and instantly react when a user plugs in an external HDMI monitor.
Because of these demanding requirements, complex drivers bypass the simple helpers
and manually construct the DRM pipeline to unlock their full capabilities.

## What's Next

If you want to dive deeper into the code, your first stop should be the [Linux GPU Driver Developer's Guide](https://www.kernel.org/doc/html/v4.18/gpu/index.html#).
This is the official source of truth and is maintained by the kernel community.
It covers everything from the basic theory of mode setting to the specific functions
you need to call to register a new driver. This documentation is the best starting point
if you want to understand the fine details of the API.

To see where the real development happens, you should follow the dri-devel mailing list.
This is the public forum where kernel engineers post their patches and debate technical changes.
You can [subscribe to the list](https://lists.freedesktop.org/mailman/listinfo/dri-devel)or browse the [archives](https://lore.kernel.org/dri-devel/) to see how experts review code.

When you feel ready to look at actual source code, do not start with the massive drivers for Intel or AMD cards.
Instead, explore the [tiny DRM drivers](https://github.com/torvalds/linux/tree/master/drivers/gpu/drm/tiny) located in the kernel source tree.
These drivers are designed for simple hardware and are small enough to read in a single sitting.
They provide a clear template for how to implement the DRM pipeline
without the massive complexity found in desktop GPU drivers.

Finally, if you want a great hands-on project to get your feet wet, take a look at the [DRM TODO list](https://dri.freedesktop.org/docs/drm/gpu/todo.html).
The upstream community is actively working on converting older, legacy fbdev drivers over to modern DRM alternatives.
This is an incredible opportunity to make a meaningful open-source contribution,
but you must deeply understand and respect [kernel maintenance ethics](https://docs.kernel.org/process/submitting-patches.html) if you decide to get involved.

Submitting patches to the Linux kernel is a collaborative process built on mutual respect for the maintainers' time.
Never send automated or untested refactoring patches, and always verify your changes on actual physical hardware before posting.
Taking the time to structure your commits cleanly and handling feedback with humility matters just as much as the code itself.

This article intentionally kept its scope narrow to give you a solid mental model of the pipeline,
but there are several concepts worth exploring once you are comfortable with the basics like
[DMA fences](https://www.reddit.com/r/linux4noobs/comments/pmf458/what_is_dma_fence/), framebuffer objects, dumb buffers, and atomic KMS.
