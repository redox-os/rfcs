- Feature Name: Video and Media Entity API (Video for Redox, V4R)
- Start Date: 2025-05-23
- RFC PR: (leave this empty)
- Redox Issue: (leave this empty)
<!-- This is first draft of the document, that's now a throwaway left for references temporarily -->
# Summary
[summary]: #summary

This document opens new chapter in Redox OS in relation to lying foundations for adding support for cameras or other video grabbing devices.
The document is a starting point to build into Redox features inspired by [Media subsystem kernel internal API][media-kernel-api] & [Linux Media Infrastructure userspace API][media-infra-us-api].

Working name of the whole video support in Redox is _Video for Redox_, abbreviated _V4R_.

# Motivation
[motivation]: #motivation

While Redox started adding some support for audio, there's no entry point for dealing with video devices (that's more than just CSI-2 or USB cameras). As Redox progresses with adoption to other targets than AMD64, it's important to lie right foundations for video support as _embedded systems_ (wide category for MCUs and SoCs) deal with other methods of connecting video input/output equipment than just USB and PCIe. The representation of all elements in the chain of components consisted in a single software video device in a very complex problem full of tiny details that need to play together. Starting before any attempt to add support for a video grabbing device would prevent falling into a trap of significant redesigns.

# Detailed design
[design]: #detailed-design

This is the bulk of the RFC. Explain the design in enough detail for somebody familiar
with the language to understand, and for somebody familiar with the compiler to implement.
This should get into specifics and corner-cases, and include examples of how the feature is used.

## Nomenclature

Media Entity
: A graph node representing an element in the video processing pipeline, usually represented by a subdevice.
: On Linux each graph is accessible via one of `/dev/media*` kernel devices, entities can be enumerated by `ioctl()` calls.

Video Subdevice
: Instance of a driver responsible for a stage in the video processing pipeline, usually represent piece of hardware peripheral or its configurable and controllable subsystem.
: On Linux each subdevice is accessible via one of `/dev/v4l-subdevice*` which exact name is obtained from media entities enumeration.

Media Pad
: Each subdevice is exposed via numbered media pads, were each can be either a sink or a source. Links between media pads are provided by enumeration of media entities.

Video Sink
: A video sink is a media pad that acts as one of logical inputs of a video subdevice; it can be either routable in a separate hardware or only represents configurable bus input of a hardware peripheral subsystem (including firmware, e.g. ISP).

Video Source
: A video source is a media pad that acts as one of logical outputs of a video subdevice; it can be either (in)active output bus in separate hardware or configurable bus output of a hardware peripheral subsystem (including firmware, e.g. ISP).

Video Endpoint (TODO: Media Endpoint?)
: Software device for applications to bind to for obtaining or publishing video frames data (in exceptional case non-video metadata).
: On Linux each kernel video device (endpoint) is accessible via one of `/dev/video*` were each one is linked with a source or a sink of one subdevice.

Stream
: To support some hardware dealing with multiplexed data streams (e.g. MIPI CSI-2) that are transferred on the same hardware bus, but are identified virtually. 
: This is very late addition to the Linux kernel and widely adopted examples of these don't exist at the time of writing of this document.

## Foundational schemes

The whole subsystem - with details designed in further RFCs - would be founded on two distinctive schemes:

* `/scheme/media` - provides access to media pipelines which control how entities are linked together (via sources and sinks) to provide means for below (one per single graph)
* `/scheme/video`
    * `/scheme/video/subdevice` - provide access to instances of drivers of individual media entities (intermediate elements) in all pipelines
    * `/scheme/video/endpoint` - actual software endpoints (character devices) to grab data from or pump data to

Linux equivalents for endpoints are `/dev/video*` which you `open()` or `mmap()` and use `ioctl()` to start and stop streaming.

Linux subdevices are exposed as `/dev/v4l-subdev*` (these are intermediate elements of the video pipeline), and media entities are controlled via `/dev/media*`; both heavily rely on `ioctl()` calls for userspace API.

```
/dev/media* (represents a pipeline [a pad's relationships graph])
|- /dev/v4l-subdev* (subset)
|  |- /dev/video* (bound to sink or source of a subdev)

```
<!-- provide a proper diagram -->

The same hierarchy in Redox would map to:

```
/scheme/media/<m>
|- /scheme/video/subdevice/<s>
|  |- /scheme/video/endpoint/<e>
```
<!-- provide a proper diagram -->

## Relationships

Each numbered file in a the `media` scheme has to provide API to read and configure links in the graph. Exact details of this API is to be defined in separate RFCs. Linux deals with that using the set of `ioctl()` calls, but Redox can just use different messages. Alternative for consideration is to reflect graphs's connection as symbolic links to subdevices' sinks and sources.

Each numbered file/directory in the `subdevice` scheme has to provide access to that subdevice's configuration (eg. controls) and sources and sinks settings (eg. pixel format) that can vary on input and output (eg. ISP doing conversion).

Each numbered file/directory in the `endpoint` scheme is a frame handling file tied to either the sink (for video output from user application) or to the source (for video input from user application).

```
/scheme/media/<ma>/<sa>:<pa> -> <sb>:<pb>
/scheme/video/subdevice/<sa>/<pa>/source -> ../../<sb>/<pb>/sink
/scheme/video/subdevice/<sb>/<pb>/sink -> ../../<sa>/<pa>/source
/scheme/video/endpoint/<ea> -> ../../subdevice/<sc>/<pc>/source
/scheme/video/endpoint/<eb> -> ../../subdevice/<sb>/<pb>/sink
```
Legend:
`<sw>` - subdevice W 
`<px>` - media pad X
`<my>` - media graph Y
`<ez>` - endpoint Z (_a character video device in Linux_)
where X,Y,Z are integers

### Example:

#### Variant A

Potentially incorrect, subject to redesign

```
/scheme/
 ├─ media/
 │  └─ 0/
 │     ├─ 0:0 -> 2:0
 │     ├─ 0:1 -> ? 
 │     ├─ 0:2 -> ?
 │     ├─ 0:3 -> ?
 │     ├─ 0:4 -> 1:0
 │     ├─ 0:5 -> 
 │     ├─ 0:6 -> 1:0
 │     └─ 0:7 -> 1:0
 └ subdevice/
    ├  0/
    │  ├─ driver
    │  ├─ 0/
    │  │  └─ sink -> /scheme/subdevice/0/0/source
    │  ├─ 1/
    │  │  └─ sink -> ?
    │  ├─ 2/
    │  │  └─ sink -> ?
    │  ├─ 3/
    │  │  └─ sink -> ?
    │  ├─ 4/
    │  │  └─ source -> /scheme/video/endpoint/0
    │  ├─ 5/
    │  │  └─ source -> /scheme/video/endpoint/1
    │  ├─ 6/
    │  │  └─ source -> /scheme/video/endpoint/2
    │  └─ 7/
    │     └─ source -> /scheme/video/endpoint/3
    ├─ 1/
    │  ├─ driver
    │  ├─ 0/
    │  │  └─ sink -> /scheme/video/subdevice/0/4 (TODO: multiple links?)
    │  └─ 1/
    │     └─ sink -> /scheme/video/endpoint/7
    └─ 2/
       ├─ driver
       └─ 0/
          └─ source -> /scheme/video/subdevice/0/sink
```

#### Variant B

subdevice node, device node...
metadata (e.g. IMU quaternions)

# Drawbacks
[drawbacks]: #drawbacks

* Approval and implementation means starting a very long painful journey of slowly building up details of a working video/media subsystem.

# Alternatives
[alternatives]: #alternatives

What other designs have been considered? What is the impact of not doing this?

* Create simple scheme for video related devices only and leave relationship between them only into userspace's libraries.
  \
  This method is though for DMA transfers between related peripherals or hardware links not existing in the software.

* Different structure:


# Unresolved questions
[unresolved]: #unresolved-questions

What parts of the design are still TBD?

- CIS-2 camera sensor drivers need access to I²C
- What are actual API calls to elements in these scheme?
- Where space to map streams if supported?


[media-kernel-api]: https://linuxtv.org/downloads/v4l-dvb-apis/driver-api/index.html
[media-infra-us-api]: https://linuxtv.org/downloads/v4l-dvb-apis/userspace-api/index.html

