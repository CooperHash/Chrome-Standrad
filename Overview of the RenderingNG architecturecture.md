Skip to content


Take a stroll down memory lane and celebrate #100CoolWebMoments since Chrome's first release.
Discover Dismiss
Blog
Overview of the RenderingNG architecture
Published on Monday, July 26, 2021

Chris Harrelson
Chris Harrelson
Rendering lead for Chrome

This post is a part of a series on the Chromium rendering engine. Check out the rest of the series to learn more about RenderingNG, key data structures, VideoNG, and LayoutNG.

In a previous post, I gave an overview of the RenderingNG architecture goals and key properties. This post will explain how its component pieces are set up, and how the rendering pipeline flows through them.

Starting at the highest level and drilling down from there, the tasks of rendering are:

Render contents into pixels on the screen.
Animate visual effects on the contents from one state to another.
Scroll in response to input.
Route input efficiently to the right places so that developer scripts and other subsystems can respond.
The contents to render are a tree of frames for each browser tab, plus the browser UI. And, a stream of raw input events from touch screens, mice, keyboards, and other hardware devices.

Each frame includes:

DOM state
CSS
Canvases
External resources, such as images, video, fonts, and SVG
A frame is an HTML document, plus its URL. A web page loaded in a browser tab has a top-level frame, child frames for each iframe included in the top-level document, and their recursive iframe descendants.

A visual effect is a graphical operation applied to a bitmap, such as scroll, transform, clip, filter, opacity, or blend.

#Architecture components
In RenderingNG, these tasks are split logically across several stages and code components. The components end up in various CPU processes, threads, and sub-components within those threads. Each plays an important role in achieving reliability, scalable performance and extensibility for all web content.

#Rendering pipeline structure
Diagram of the rendering pipeline as explained in the following text.
Rendering proceeds in a pipeline with a number of stages and artifacts created along the way. Each stage represents code that does one well-defined task within rendering. The artifacts are data structures that are inputs or outputs of the stages; in the diagram inputs or outputs are indicated with arrows.

This blog post won't go into much detail about the artifacts; that will be discussed in the next post: Key Data Structures and their roles in RenderingNG.

#The pipeline stages
In the preceeding diagram, stages are notated with colors indicating in which thread or process they execute:

Green: render process main thread
Yellow: render process compositor
Orange: viz process
In some cases they can execute in multiple places, depending on the circumstance, which is why some have two colors.

The stages are:

Animate: change computed styles and mutate property trees over time based on declarative timelines.
Style: apply CSS to the DOM, and create computed styles.
Layout: determine the size and position of DOM elements on the screen, and create the immutable fragment tree.
Pre-paint: compute property trees and invalidate any existing display lists and GPU texture tiles as appropriate.
Scroll: update the scroll offset of documents and scrollable DOM elements, by mutating property trees.
Paint: compute a display list that describes how to raster GPU texture tiles from the DOM.
Commit: copy property trees and the display list to the compositor thread.
Layerize: break up the display list into a composited layer list for independent rasterization and animation.
Raster, decode and paint worklets: turn display lists, encoded images, and paint worklet code, respectively, into GPU texture tiles.
Activate: create a compositor frame representing how to draw and position GPU tiles to the screen, together with any visual effects.
Aggregate: combine compositor frames from all the visible compositor frames into a single, global compositor frame.
Draw: execute the aggregated compositor frame on the GPU to create pixels on-screen.
Stages of the rendering pipeline can be skipped if they aren't needed. For example, animations of visual effects, and scroll, can skip layout, pre-paint and paint. This is why animation and scroll are marked with yellow and green dots in the diagram. If layout, pre-paint, and paint can be skipped for visual effects, they can be run entirely on the compositor thread and skip the main thread.

Browser UI rendering is not depicted directly here, but can be thought of as a simplified version of this same pipeline (and in fact its implementation shares much of the code). Video (also not directly depicted) generally renders via independent code that decodes frames into GPU texture tiles that are then plugged into compositor frames and the draw step.

#Process and thread structure
There are more CPU processes and threads in the Chromium architecture than pictured here. These diagrams focus only on those critical to rendering.

#CPU processes
The use of multiple CPU processes achieves performance and security isolation between sites and from browser state, and stability and security isolation from GPU hardware.

Diagram of the various parts of the CPU processes
The render process renders, animates, scrolls, and routes input for a single site and tab combination. There are many render processes.
The browser process renders, animates, and routes input for the browser UI (including the URL bar, tab titles and icons), and routes all remaining input to the appropriate render process. There is exactly one browser process.
The Viz process aggregates compositing from multiple render processes plus the browser process. It rasters and draws using the GPU. There is exactly one Viz process.
Different sites always end up in different render processes. (In reality, always on desktop; when possible on mobile. I'll write "always" below, but this caveat applies throughout.)

Multiple browser tabs or windows of the same site usually go in different render processes, unless the tabs are related (one opening the other). Under strong memory pressure on desktop Chromium may put multiple tabs from the same site into the same render process even if not related.

Within a single browser tab, frames from different sites are always in different render processes from each other, but frames from the same site are always in the same render process. From the perspective of rendering, the important advantage of multiple render processes is that cross-site iframes and tabs achieve performance isolation from each other. In addition, origins can opt into even more isolation.

There is exactly one Viz process for all of Chromium. After all, there is usually only one GPU and screen to draw to. Separating Viz into its own process is good for stability in the face of bugs in GPU drivers or hardware. It's also good for security isolation, which is important for GPU APIs like Vulkan. It's also important for security in general.

Since the browser can have many tabs and windows, and all of them have browser UI pixels to draw, you might wonder: why there is exactly one browser process? The reason is that only one of them is focused at a time; in fact, non-visible browser tabs are mostly deactivated and drop all of their GPU memory. However, complex browser UI rendering features are increasingly being implemented in render processes as well (known as WebUI). This is not for performance isolation reasons, but in order to take advantage of the ease of use of Chromium's web rendering engine.

On older Android devices, the render and browser process are shared when used in a WebView (this does not apply to Chromium on Android generally, just WebView). On WebView, the browser process is also shared with the embedding app, and WebView has only one render process.

There is also sometimes a utility process for decoding protected video content. This process is not depicted above.

#Threads
Threads help achieve performance isolation and responsiveness in spite of slow tasks, pipeline parallelization and multiple buffering.

A diagram of the render process as described in the article.
The main thread runs scripts, the rendering event loop, the document lifecycle, hit testing, script event dispatching, and parsing of HTML, CSS and other data formats.
Main thread helpers perform tasks such as creating image bitmaps and blobs that require encoding or decoding.
Web Workers run script, and a rendering event loop for OffscreenCanvas.
The Compositor thread processes input events, performs scrolling and animations of web content, computes optimal layerization of web content, and coordinates image decodes, paint worklets and raster tasks.
Compositor thread helpers coordinate Viz raster tasks, and execute image decode tasks, paint worklets and fallback raster.
Media, demuxer or audio output threads decode, process and synchronize video and audio streams. (Remember that video executes in parallel with the main rendering pipeline.)
Separating the main and compositor threads is critically important for performance isolation of animation and scrolling from main thread work.

There is only one main thread per render process, even though multiple tabs or frames from the same site may end up in the same process. However, there is performance isolation from work performed in various browser APIs. For example, generation of image bitmaps and blobs in the Canvas API run in a main thread helper thread.

Likewise, there is only one compositor thread per render process. It is generally not a problem that there is only one, because all of the really expensive operations on the compositor thread are delegated to either compositor worker threads or the Viz process, and this work can be done in parallel with input routing, scrolling or animation. Compositor worker threads coordinate tasks run in the Viz process, but GPU acceleration everywhere can fail for reasons outside of Chromium's control, such as driver bugs. In these situations the worker thread will do the work in a fallback mode on the CPU.

The number of compositor worker threads depends on the capabilities of the device. For example, desktops will generally use more threads, as they have more CPU cores and are less battery-constrained than mobile devices. This is an example of scaling up and scaling down.

It's also interesting to note that the render process threading architecture is an application of three different optimization patterns:

Helper threads: sending long-running subtasks off to additional threads, to keep the parent thread responsive to other requests happening simultaneously. The main thread helper and compositor helper threads are good examples of this technique.
Multiple buffering: showing previously rendered content while rendering new content, to hide the latency of rendering. The compositor thread uses this technique.
Pipeline parallelization: running the rendering pipeline in multiple places simultaneously. This is how scrolling and animation can be fast, even if a main thread rendering update is happening, because scroll and animation can run in parallel.
#Browser process
A browser process diagram showing the relationship between the Render and compositing thread, and the render and compositing thread helper.
The render and compositing thread responds to input in the browser UI, routes other input to the correct render process; lays out and paints browser UI.
The render and compositing thread helpers execute image decode tasks and fallback raster or decode.
The browser process render and compositing thread are similar to the code and functionality of a render process, except that the main thread and compositor thread are combined into one. There is only one thread needed in this case because there is no need for performance isolation from long main thread tasks, since there are none by design.

#Viz process
A diagram showing that the Viz process includes the GPU main thread, and the display compositor thread.
The GPU main thread rasters display lists and video frames into GPU texture tiles, and draws compositor frames to the screen.
The display compositor thread aggregates and optimizes compositing from each render process, plus the browser process, into a single compositor frame for presentation to the screen.
Raster and draw generally happen on the same thread, because both of them rely on GPU resources, and it's hard to reliably make multi-threaded use of the GPU (easier multi-threaded access to the GPU is one motivation for developing the new Vulkan standard). On Android WebView, there is a separate OS-level render thread for drawing because of how WebViews are embedded into a native app. Other platforms will likely have such a thread in the future.

The display compositor is on a different thread because it needs to be responsive at all times, and not block on any possible source of slowdown on the GPU main thread. One cause of slowdown on the GPU main thread is calls into non-Chromium code, such as vendor-specific GPU drivers, that may be slow in hard-to-predict ways.

#Component structure
Within each render process main or compositor thread, there are logical software components that interact with each other in structured ways.

#Render process main thread components
A diagram of the Blink renderer.
Blink renderer:
The local frame tree fragment represents the tree of local frames and the DOM within frames.
The DOM and Canvas APIs component contains implementations of all of these APIs.
The document lifecycle runner executes the rendering pipeline steps up to and including the commit step.
The input event hit testing and dispatching component executes hit tests to find out which DOM element is targeted by an event, and runs the input event dispatching algorithms and default behaviors.
The rendering event loop scheduler and runner decides what to run on the event loop and when. It schedules rendering to happen at a cadence matching the device display.
A diagram of the frame tree.
Local frame tree fragments are a bit complicated to think about. Recall that a frame tree is the main page and its child iframes, recursively. A frame is local to a render process if it is rendered in that process, and otherwise it is remote.

You can imagine coloring frames according to their render process. In the preceding image, the green circles are all frames in one render process; the red ones are in a second, and the blue one is in a third.

A local frame tree fragment is a connected component of the same color in a frame tree. There are four local frame trees in the image: two for site A, one for site B, and one for site C. Each local frame tree gets its own Blink renderer component. A local frame tree's Blink renderer may or may not be in the same render process as other local frame trees (it's determined by the way render processes are selected, as described earlier).

#Render process compositor thread structure
A diagram showing the render process compositor components.
The render process compositor components include:

A data handler that maintains a composited layer list, display lists and property trees.
A lifecycle runner that runs the animate, scroll, composite, raster, and decode and activate steps of the rendering pipeline. (Remember that animate and scroll can occur in both the main thread and the compositor.)
An input and hit test handler performs input processing and hit testing at the resolution of composited layers, to determine if scrolling gestures can be run on the compositor thread, and which render process hit tests should target.
#An example in practice
Let's now make the architecture concrete with an example. In this example there are three tabs:

Tab 1: foo.com

<html>
  <iframe id=one src="foo.com/other-url"></iframe>
  <iframe  id=two src="bar.com"></iframe>
</html>
Tab 2: bar.com

<html>
 ???
</html>
Tab 3: baz.com

<html>
 ???
</html>
The process, thread and component structure for these tabs will look like this:

Diagram of the process for the tabs.
Now let's walk through one example each of the four main tasks of rendering, which as you may recall are:

Render contents into pixels on the screen.
Animate visual effects on the contents from one state to another.
Scroll in response to input.
Route input efficiently to the right places so that developer scripts and other subsystems can respond.
To render the changed DOM for tab one:


A developer script changes the DOM in the render process for foo.com.
The Blink renderer tells the compositor that it needs a render to occur.
The compositor tells Viz it needs a render to occur.
Viz signals the start of the render back to the compositor.
The compositor forwards the start signal on to the Blink renderer.
The main thread event loop runner runs the document lifecycle.
The main thread sends the result to the compositor thread.
The compositor event loop runner runs the compositing lifecycle.
Any raster tasks are sent to Viz for raster (there are often more than one of these tasks).
Viz rasters content on the GPU.
Viz acknowledges completion of the raster task. Note: Chromium often doesn't wait for the raster to complete, and instead uses something called a sync token that has to be resolved by raster tasks before step 15 executes.
A compositor frame is sent to Viz.
Viz aggregates the compositor frames for the foo.com render process, the bar.com iframe render process, and the browser UI.
Viz schedules a draw.
Viz draws the aggregated compositor frame to the screen.
To animate a CSS transform transition on tab two:


The compositor thread for the bar.com render process ticks an animation in its compositor event loop by mutating the existing property trees. This then re-runs the compositor lifecycle. (Raster and decode tasks may occur, but are not depicted here.)
A compositor frame is sent to Viz.
Viz aggregates the compositor frames for the foo.com render process, the bar.com render process, and the browser UI.
Viz schedules a draw.
Viz draws the aggregated compositor frame to the screen.
To scroll the web page on tab three:


A sequence of input events (mouse, touch or keyboard) come to the browser process.
Each event is routed to baz.com's render process compositor thread.
The compositor determines if the main thread needs to know about the event.
The event is sent, if necessary, to the main thread.
The main thread fires input event listeners (pointerdown, touchstar, pointermove, touchmove or wheel) to see if listeners will call preventDefault on the event.
The main thread returns whether preventDefault was called to the compositor.
If not, the input event is sent back to the browser process.
The browser process converts it to a scroll gesture by combining it with other recent events.
The scroll gesture is sent once again to baz.com's render process compositor thread,
The scroll is applied there, and the compositor thread for the bar.com render process ticks an animation in its compositor event loop. This then mutates scroll offset in the property trees and re-runs the compositor lifecycle. It also tells the main thread to fire a scroll event (not depicted here).
A compositor frame is sent to Viz.
Viz aggregates the compositor frames for the foo.com render process, the bar.com render process, and the browser UI.
Viz schedules a draw.
Viz draws the aggregated compositor frame to the screen.
Note that each input event after the first can skip steps three and four, because the scroll has already begun and at that point scripts can observe it via the scroll event, but no longer interrupt.

To route a click event on a hyperlink in iframe #two on tab one:


An input event (mouse, touch or keyboard) comes to the browser process. It performs an approximate hit test to determine that the bar.com iframe render process should receive the click, and sends it there.
The compositor thread for bar.com routes the click event to the main thread for bar.com and schedules a rendering event loop task to process it.
The input event processor for bar.com's main thread hit tests to determine which DOM element in the iframe was clicked, and fires a click event for scripts to observe. Hearing no preventDefault, it navigates to the hyperlink.
Upon load of destination page of the hyperlink, the new state is rendered, with steps similar to the "render changed DOM" example above. (These subsequent changes are not depicted here.)
#Conclusion
Phew, that was a lot of detail. As you can see, rendering in Chromium is quite complicated! It can take a lot of time to remember and internalize all the pieces, so don't be worried if it seems overwhelming.

The most important takeaway is that there is a conceptually simple rendering pipeline, which through careful modularization and attention to detail, has been split into a number of self-contained components. These components have then been split across parallel processes and threads to maximize scalable performance and extensibility opportunities.

Each of those components plays a critical role in enabling all of the performance and features modern web apps need. Soon we'll publish deep-dives into each of them, and the important roles they play.

But before that, I'll also explain how the key data structures mentioned in this post (the ones indicated in blue at the sides of the rendering pipeline diagram) are just as important to RenderingNG as code components.

Thanks for reading, and stay tuned!

Illustrations by Una Kravets.

Rendering
Last updated: Monday, July 26, 2021 ??? Improve article

Follow us
Contribute
File a bug
View source
Related content
web.dev
Web Fundamentals
Case studies
Podcasts
Connect
Twitter
YouTube
GitHub
Chrome
Firebase
All products
Privacy
Terms
Choose language
ENGLISH (en)
Content available under the CC-BY-SA-4.0 license
