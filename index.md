# The Rapid Analysis Framework and its relationship to RubinTV

## Preamble (aka. soapbox)

> This technote is both a work-in-progress, because
> * 1) it is a provisional version which won't be tidied up until after the design review
> * 2) it is documenting a moving target, meaning that there are places where the present tense is used to save having to write everything twice, but where the work hasn't quite been done yet. These are, however, all places where the work is planned and underway, _i.e._ it's not aspirational, it's just not quite finsihed yet, but will be on the scale of weeks.

> There are many ways to skin a cat. This is how it is _currently_ being done, but nothing in here should be taken as an assertion that this is how it _should_ be being done, merely that this is how it is at present (and that without significant intervention and extra effort allocated, it will remain so, due to the finite nature of the allocated effort and enormous scope of remaining work).

> Your forgiveness is requested: this would not have been written alone/without oversight if I had been given some help/oversight, but as we all know, the project is under-resourced, and this is what happened. Nobody is to blame, but the code we have is what I came up with because I had to to make things work for those on the summit.

> Finally, it is perhaps worth remembering that the initial architecure was conceived and deployed a) organically, as it was required, and b) that it came into being in January 2020 with the delivery of LATISS to the summit, so if you find youself thinking "why doesn't it look like/use X", recall what the DM landscape was back then, and what existed in terms of leverageable architectures. Given that, I think it's quite surprising how _not_ bad it all is! YMMV though, of course...

## Abstract

```{abstract}
Documentation on the architecture of the Rapid Analysis Framework, including its relationship to RubinTV.
```

## Introduction

Historically, there has been confusion about what Rapid Analysis is, what RubinTV is, and what the relationship and boundaries are between the two. This is because they were developed at the same time, co-evolving with one another. However, they are entirely separate, and totally decoupled, in the technical sense. Therefore, we begin with definitions of each, paying careful attention to say exactly what each is, and is not. This technote will provide a detailed description of, and motivation for, the Rapid Analysis Framework, but does not go into depth about the workings of RubinTV - that will be covered in enough depth to make it clear what it is and where the boundaries are, but no more than that. It is also worth nothing that some of this confusion came from some poor naming choices made as the co-evolution of Rapid Analysis and RubinTV was ongoing, almost all of which have now been rectified, with the only item remaining to be fixed being that the RA framework still lives in a repo called `rubintv_production`. (XXX can I fix this before the review?! Probably not without risking breakage of the summit, especially as some people might well still be on vacation)

#### What Rapid Analysis is/isn't

Rapid Analysis (RA) is a realtime processing framework. It is a set of kubernetes pods which run in various locations, and process data in realtime. Currently, these locations are:
* On the summit
* USDF
* BTS (for bi-directional testing)
* TTS (for bi-directional testing)

RA is _not_ deployed via phalanx. It is administered via ArgoCD, and the repo which defines the pods and their memory/CPU allocations is [link to repo](https://github.com/lsst-ts/argocd-csc/). XXX update that link to point to the rapid_analysis part.

Rapid Analysis has several outputs:
* it `butler.put()`s its outputs to the relevant local repo
* it (will) output to the Visit/Summit database
* it sends files to local S3 storage so that data can be displayed on RubinTV

#### What RubinTV is/isn't

RubinTV is a website. It is entirely decoupled from Rapid Analysis, with the data flow (thus far) being strictly one-way, from RA to RubiNTV. However, to date, everything displayed on RubinTV has been processed and uploaded via RA, but that there is nothing that necessarily makes that the case, and in plans exist for significant extra display scope via RubinTV that will not have been produced (directly) by RA.

Phalanx - simply because we need auth for embargo compliance.

There is two RubinTV sites for each location, a dev and a prod site.

#### What are `summit_utils` and `summit_extras`?

A quick note here about the packages, why they lives in [the SITCOM GitHub org](https://github.com/lsst-sitcom/) and not in the `lsst_distrib` metapackage, but instead in `lsst_sitcom`.

These packages break telescope agnosticism, and thus are not allowed in the core DM/Science Pipelines codebase. I initially pushed for this, but understood the point that this code simply does not belong there. It also contains code which people don't like/object to (often because it's not pipeline-like), but which needs to exist regardless. It also contains a lot a lot of syntactic sugar/one-line-makers to make things easier for summit folk (who are not pipeline devs).

In `summit_utils`, the test coverage is excellent, nothing is added without adding tests for it, and these tests are all in Jenkins via either either unit tests, or via the `ci_summit` package (which runs as part of the nightly build, and on request). In some ways, code here should be more carefully tested and bugs/breakages considered worse than in Science Pipelines (if such a thing is permissable to say!). This is because problems here can directly result in lost telescope time or observing inefficiencies, as code in here is relied up by the scheduler and other closed-loop T&S operational code.

The `summit_extras` package, however, is not like this. It is a place for adding useful code which should live in one place (for all the obvious reasons - passing functions around notebooks is Very Bad, we can all agree) but which should explicitly not be relied upon.


## Why Rapid Analysis `!=` Prompt Processing Framework `!=` OCPS

> Soapbox section: First and foremost, Rapid Analysis is a pragmatic, "let's say yes/make things happen" arena. That's an odd things to say though, so why am I saying that? There are a large number of brilliant, hardworking scientists working on the summit to commission the system, and many of them are frequently blocked by needing various thigns to exist which simply weren't planned (at least with respect to data processing), and which, if we stopped to do things the way we'd all like them to be done, would results in significant (multiple months) of critical path deplays(_i.e._ this is real, important stuff, and saying "we should have done it like X" is definitely a valid point, but _needs_ to price in "would you still say that if someone said it was a critical path activity and doing it that way would delay the activity by ~2 months, and in more than one instance?".

* Many things are not (and could not be) `pipelineTasks`:
    * Data not ingested - working on raw files, _i.e._ no `butler` involvment
        * StarTracker data, all sky camera images, `TMAEvents`, catchup services
    * Contacting the EFD
    * Writing to _and reading from_ the visit/summit database
* Speed considerations
    * python `import lsst.*` spinup time
    * quantum graph generation delay
    * calib caching
* Needs to work in the absence of `next_visit` events, as that won't always happen during commissioning (at least not with enough notice to do it the PP way) (XXX need to check this is true with KT)
* Need to control behaviour when we can't keep up
    * Summit compute is not elastic the way PP is


## The Services

The overwhleming majority of Rapid Analysis pods will, once the main camera is taking data, be for processing the full focal plane data. However, when counting by type or by code line-count[^podcount], the vast majority of the Rapid Analysis deployed services are actually "snowflake services" - realtime processing services for things which have been written from scratch to support a specific case[^refactor].

[^podcount]: And for now, because LSSTCam is not yet on sky, by count too, but that will, of course, not remain the case forever.
[^refactor]: It is quite possible that a _little_ of this could be refactored now, but doing that before now would have been very premature, and actually may well still be, or may simply not be worth the effort.

### Snowflake Services

##### All sky images
A service (outside of RA) writes files to a fixed root path on disk at the summit, with the subdirectories named by the `dayObs`. This services monitors path for new per-day directories, and once they exist, scans for new files, and each time one lands, restretches the PNG file to enhance the contrast, overlays the cardinal directions on it, labels it for the time it acquired, and send to S3 for diplay on RubinTV. Every 10 minutes, all images on that day are animated and the updated animation is also sent to S3 for RubinTV display. These images are not ingested, and an obs_package does not exist for them.

##### StarTracker Image Processing

##### Monitor Image Serving

For each new LATISS image, this service creates a `.png` of the post-ISR focal plane and sends it to S3 for display on RubinTV. Also pulls metadata from the image, and sends it to the LATISS `MetadataServer` for dispatch to RubinTV (and once it exists, the Summit/Visit Database).

##### Metadata Servers

##### Night Report Generation

##### Catchup Service

##### ImageExaminer (likely needs replacing)

For each new LATISS image, this service creates a `.png` containg some very quick canned analyses which are appropriate to run on all LATISS images, and sends it to S3 for display on RubinTV.

##### SpectrumExaminer

##### AuxTel Mount Torque Analysis

##### TMA Event Generation

A `TMAEventMaker` is run in a loop, such that every time the TMA moves, new events are geneated from the EFD via the `TMAStateMachine` (see [SITCOMTN-098](https://sitcomtn-098.lsst.io/) for the technical details on `TMAEvent` generation). For each new TMA movement, various bits of telemetry are pulled from the EFD, and plots and metrics are created from them, and are sent to RubinTV for display (and will be sent to the Summit/Visit Database once it is possible).


### Full Focal Plane Processing

For the sake of simplicity, this section will only consider the summit, but note that simialar systems are in place at all locations, with no real architectural differences, just different `instruments` and data streams being processed.

For each `instrument` (which on the summit will be, at some point, likely all of `LATISS`, `LSSTComCam` and `LSSTCam` simultaneously), the architecture is as follows:

A head node exists, which watches for new data landing in the instrument's repo, and dispatches this in a dynamic and configurable way to the worker nodes, as detailed in [Processing Control](#processing-control).

Scatter/gather.
* ISR
* SFM

#### Processing control {#processing-control}

dynamic and configurable: dynamic based on the data rate and how we're keeping up. Configurable via LOVE and notebooks (for a _very_ select set of power users)

{images of focal plane patterns here}

Side notes:

## `git-pull` on pod start

- Motivation
- How it works
    - list of packages
- responsibility.


## Techdebt

There are several places where Rapid Analysis has significant tech debt. Some of it is minor, but ugly and just in need of tidyup because it was done in a hurry, but some of it is deeper and a risk to ongoing stability and maintainance, and hinders developement.

* Test coverage - both unit tests and CI testing:
The coverage is frankly shocking. It not only makes development harder and more scary, but hinders larger, more refactoring-like work, and thus causes a vicious circle, with the debt compounding as the lack of tests makes tidying up harder itself.
* `LocationConfig`:
This is honestly just pretty gross. It is not a hard fix, but it does need doing.
* Package (and some file) renaming
Finally deal with the hangover of `rubintv_production` etc.
* The RA pod build system and devOps work
It may well be that there's no real tech debt here, it may just need more FTEs. Basically it needs to be so that it doesn't need to be done me and Tiago


Some of the channels are just a bit hacky and gross
They're only the low priority ones though, e.g. all sky camera etc

