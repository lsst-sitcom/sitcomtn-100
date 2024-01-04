# The Rapid Analysis Framework and its relationship to RubinTV

```{abstract}
Documentation on the architecture of the Rapid Analysis Framework, including its relationship to RubinTV.
```

## Preamble (aka. Merlin's soapbox)

> This technote is both a work-in-progress, because
> * 1) it is a provisional version which won't be tidied up until after the design review
> * 2) it is documenting a moving target, meaning that there are places where the present tense is used to save having to write everything twice, but where the work hasn't quite been done yet. These are, however, all places where the work is planned and underway, _i.e._ it's not aspirational, it's just not quite finsihed yet, but will be on the scale of weeks.

> There are many ways to skin a cat. This is how it is _currently_ being done, but nothing in here should be taken as an assertion that this is how it _should_ be being done, merely that this is how it is at present (and that without significant intervention and extra effort allocated, it will remain so, due to the finite nature of the allocated effort and enormous scope of remaining work).

> Your forgiveness is requested: this would not have been written alone/without oversight if I had been given some help/oversight, but as we all know, the project is under-resourced, and this is what happened. Nobody is to blame, but the code we have is what I came up with because I had to to make things work for those on the summit.

> Finally, it is perhaps worth remembering that the initial architecure was conceived and deployed a) organically _i.e._ as it was required, and b) that it came into being in January 2020 with the delivery of LATISS to the summit, so if you find youself thinking "why doesn't it look like/use X", recall what the DM landscape was back then, and what existed in terms of leverageable architectures. Given that, I think it's quite surprising how _not_ bad it all is. Of course, YMMV...

## Introduction

Historically, there has been confusion about what Rapid Analysis is, what RubinTV is, and what the relationship and boundaries are between the two. This is because they were developed at the same time, co-evolving with one another. However, they are entirely separate, and totally decoupled, in the technical sense. Therefore, we begin with definitions of each, paying careful attention to say exactly what each is, and is not. This technote will provide a detailed description of, and motivation for, the Rapid Analysis Framework, but does not go into depth about the workings of RubinTV - that will be covered in enough depth to make it clear what it is and where the boundaries are, but no more than that. It is also worth nothing that some of this confusion came from some poor naming choices made as the co-evolution of Rapid Analysis and RubinTV was ongoing, almost all of which have now been rectified, with the only item remaining to be fixed being that the RA framework still lives in a repo called `rubintv_production`. (XXX can I fix this before the review?! Probably not without risking breakage of the summit, especially when some people might still be on vacation)

### What Rapid Analysis is/isn't

Rapid Analysis (RA) is a realtime processing framework. It is a set of kubernetes pods and a work distribution system which runs in various locations, processing data in realtime. Currently, these locations are:
* On the summit
* USDF
* BTS (for bi-directional testing)
* TTS (for bi-directional testing)

RA is _not_ deployed via phalanx. It is administered via ArgoCD, and the repo which defines the pods and their memory/CPU allocations is [here](https://github.com/lsst-ts/argocd-csc/). XXX update that link to point to the rapid_analysis part.

Rapid Analysis has several outputs:
* it `butler.put()`s its outputs to the relevant local repo
* it (will) output to the Visit/Summit database
* it sends metrics to Sasquatch
* it sends files to local S3 storage so that data can be displayed on RubinTV

What it is not:
* it has no frontend
* it does no display of anything
    * In the future it will display its health and processing status in LOVE.

Because it does render pngs, and sends them to RubinTV for display, the package used to be called `rubintv_production`, but is in the process of being renamed to `rapid_analysis` (which, ironically, is what it was called originally, before I was told to rename it).

The processing philosophy of RA is to process things as quickly as possible, and when unable to keep up (for whatever reason), to only process the most recent image for a given data stream, _i.e._ to prioritise staying present and leaving gaps over complete processing. Some catchup services exist, on a best-effort basis, and to-date they have not saturated, but no guarantees are made with respect to completeness of processing. Details on how this is handled, and how it will work for full focal plane processing are given later.

### What RubinTV is/isn't

RubinTV is a python web app deployed via Phalanx. It is entirely decoupled from Rapid Analysis, with the data flow (thus far) being strictly one-way, from RA to RubinTV[^rubintv_queries]. To date, everything displayed on RubinTV has been processed and uploaded via RA, but that there is nothing that necessarily makes that the case. There are for significant plans for data display (the Derived Data Visualiszation work) that will use RubinTV as frontend, and that will display data which will not have been produced by RA, _e.g._ EFD data.

[^rubintv_queries]: The Derived Data Visualiszation work will involve user-generated queries being sent to backend databases, meaning that RubinTV will no longer strictly be scraping-only, though I still don't think it will _write_ anything.

The RubinTV web app code [lives here](https://github.com/lsst-ts/rubintv/), and has been officially adopted by T&S. It is now deployed via Phalanx, which is because we need authentication for embargo compliance.

There are two RubinTV sites for each location, a dev and a prod site.

What is is not:
* RubinTV does no processing of anything
* Once the Derived Data Visualisation code is integrated, it will retrieve data for plotting from:
    * The butler
    * The EFD
    * The Summit/Visit Database

#### The history (and thus the shape) of RubinTV
This section will probably be deleted, but just to help with the review:

Simon helped us put together the initial web app, and thus it was born in the SQuaRE org, and used `saphir`. When it needed to move beyond a web 1.0 hand-crafted html design, I employed Guy Whittaker as a web dev so that I didn't have to learn how to make websites. This initial version was deployed on Rounttable, and stored all its data in GCS. The summit data lives in the `rubintv_data` GCS bucket, with `rubintv_data_tts` and `rubintv_data_usdf` existing to store data from the TTS and USDF respectively. All these were served from a single frontend, which had various locations selectable from the front page, which then told the app which bucket's data to display.

When it became clear that RubinTV was going to live long term, and that its contents would be subject to embargo compliance, v2 was written, which brought about several changes. Firstly, it moved from Roundtable deployment to phalanx. It changed frameworks from `saphir` to `FastAPI`, and moved from GCS storage to being S3 backed, so that we could self-host the data for embargo reasons. This move to S3, plus the need to have the summit display be resilient to network outages, necessitated a change in how the web app is deployed and backed. It is therefore now: one S3 bucket per (non-USDF) location, with two web app instances (dev and prod), with each web app hosting the data from its own location, _i.e._ the summit instance shows the real summit data, the TTS deployment only shows emulated data from the TTS, and likewise the BTS shows emulated BTS data only, too. The USDF-hosted web app breaks the pattern, by being the superset of all locations: there are four S3 buckets at USDF: three which are mirrors of the other buckets (summit, TTS and BTS), and a local bucket containing USDF's own data (data created from cleanroom testing, which continue being generated at USDF). The USDF web apps therefore still have a selectable location, whereas the per-location sites will not, as they only have access to their local data.

XXX diagram of buckets here.

#### What are `summit_utils` and `summit_extras`?

A quick note here about the packages, why they live in [the SITCOM GitHub org](https://github.com/lsst-sitcom/) and not in the `lsst_distrib` metapackage, but instead in `lsst_sitcom`.

These packages break telescope agnosticism, and thus are not allowed in the core DM/Science Pipelines codebase. I initially pushed for this, but understood the point that this code simply does not belong there. It also contains code which people don't like/object to (often because it's not pipeline-like), but which needs to exist, regardless. It also contains a lot a lot of syntactic sugar/one-line-makers to make things easier for summit folk (who are not (pipeline) devs).

In `summit_utils`, the test coverage is excellent, and nothing should be added without adding tests coverage. These tests are all in Jenkins via either either unit tests (where possible), or via the `ci_summit` package when unit testing isn't possibly. `ci_summit` runs as part of the nightly build, as well as being available on request. In some sense, bugs and breakages here should be considered worse than in Science Pipelines (if such a thing is permissable to say!), and code should therefore be as, or even _more_, carefully tested. This is because problems here can directly result in lost telescope time or observing inefficiencies, as code in here is relied upon by the scheduler and other closed-loop T&S operational code.

The `summit_extras` package, however, is not like this. It is a place for adding useful code which should live in one place (for all the obvious reasons - passing functions around notebooks is Very Bad, I'm sure we can all agree) but which should explicitly not be relied upon for anything important, operational of scientific[^summit_extras_considered_evil?].

[^summit_extras_considered_evil?]: I am certain that some people would argue that the set of code which falls into this category _should be_ empty, because it either should be well-designed and tested and work for all use-cases, or it shouldn't exist, but I am not one of those people.


## Why Rapid Analysis `!=` Prompt Processing Framework `!=` OCPS

> Soapbox section: First and foremost, Rapid Analysis is a pragmatic, "let's say yes/make things happen" arena. That's an odd things to say though, so why am I saying that? There are a large number of brilliant, hardworking scientists working on the summit/elsewhere to commission the system, and many of them are frequently blocked by needing various thigns to exist which simply weren't planned (at least with respect to data processing and software support), and which, if we stopped to do things the way we'd all like them to be done, would result in significant (== multiple months) of critical path deplays (_i.e._ this is real, important stuff, and saying "we should have done it like X" is definitely a valid point, but _needs_ to price in "would you still say that if someone said it was a critical path activity and doing it that way would delay the activity by ~2 months, and in more than one instance?").

The overwhleming majority of Rapid Analysis pods will (once the main camera is taking data) be for processing the full focal plane data via Single Frame Measurement. However, when counting by either type or by code line-count[^podcount], the vast majority of the Rapid Analysis deployed services are actually "snowflake services" - realtime processing services for things which have been written from scratch to support a specific non-SFM use-case[^refactor].

[^podcount]: And for now, because LSSTCam is not yet on sky, by count too, but that will, of course, not remain the case forever.
[^refactor]: It is quite possible that a _little_ of this could be refactored now, but doing that before now would have been very premature, and actually may well still be, or may simply not be worth the effort.

* Many things are not (and could never be) `pipelineTasks`:
    * Some data is not ingested - in these cases RA is working on raw files, _i.e._ there is no `butler` involvment. Examples include:
        * StarTracker data
        * All sky camera images
        * `TMAEvents`
        * More to come, I am sure...
    * Contacting the EFD
    * Writing to _and reading from_ the visit/summit database
* Speed considerations:
    * Starting python and executing `import lsst.*` spinup time (XXX time this and give number)
    * Quantum graph generation delay (used to be significant, now may be OK, I haven't timed it recently, but calibs used to make things bad even for trivial output graphs)
    * Calib caching in the running process (XXX time this and give number)
* Needs to work in the absence of `next_visit` events, as that won't always happen during commissioning (at least not with enough notice to do it the PP way) (XXX need to check this is true with KT - does that come from OCS/scheduler, _i.e._ do we get one for CCS image or calibs etc)
* We need to control the behaviour when we can't keep up
    * Summit compute is not elastic the way PP is, we downsize processing when images are too fast, whereas PP expands to deal with demand.


## Rapid Analysis Services

The [Snowflake services](#snowflake-services) are all services which have been written from scratch for RA and are not just thin wrappers around SFM. The SFM pipeline pods are covered in the [Full Focal Plane Processing](#processing-control) section, which covers LSSTCam, LSSTComCam and LATISS SFM processing (LATISS, when taking survey images, is also "full focal plane processing", it just has a single chip).

(snowflake-services)=
### Snowflake Services

* All sky images
A service (outside of RA) writes files to a fixed root path on disk at the summit, with the subdirectories named by the `dayObs`. The all sky services monitors this path for new per-day directories, and once they exist, scans for new files, and each time one lands, restretches the PNG file to enhance the contrast, overlays the cardinal directions on it, overlays the time the image was taken, and sends it to S3 for diplay on RubinTV, as well as uploading to the EPO GCS bucket for public display. Every 10 minutes, all images on taken on current day are animated, and the updated animation is sent to S3 for RubinTV display. These images are not ingested, and an obs_package does not exist for them, nor is there one planned, as far as I am aware. At the end of the day the temporary files are deleted and the final animation is sent to S3 for access in RubinTV in the all sky "historical" section for posterity.

#### StarTracker Image Processing
For every image taken by the three Star Tracker cameras (wide, narrow and fast) a `wcs` is constructed from the image headers (including fixups), source detection is run, and an astrometric solution is found via astrometry.net. The offset between the fitted `wcs` and the nominal TMA pointing is calculated, and the various wcs numbers and residuals are sent to RubinTV for display. These images are not ingested, and a working obs_package does not exist for them.[^startracker_obs_package]

[^startracker_obs_package]: Once it does, image retreival and wcs creation could be done via a butler and `afwImage`, but this does not really gain RA much, though it would make some people happy. However, despite being this being "imminent", I've been waiting ~18 months (XXX check when first StarTracker data was on RubinTV and update this number) for this to happen, and there has been no progress on this for a long time now.

#### Monitor Image Serving

For each new LATISS image, this service creates renders a `.png` of the post-ISR focal plane and sends it to S3 for display on RubinTV. It also pulls metadata from the image, and sends it to the LATISS `MetadataServer` for dispatch to RubinTV (and once it exists, the Summit/Visit Database).

#### Metadata Servers

XXX Write this section

#### Night Report Generation

This is a service which creates a selection of pre-defined plots throughout the night. Each time a new image is taken, all the plots are recreated and sent to RubinTV. There is an small framework which makes adding new plots very easy so that summit users/commissioning folk can easily add plots here to appear on RubinTV. It is possible that this will be depracated/removed once the Derived Data Visualization work is done, but that remains to be seen, as it is possible there will still be a reason to have these made in a fixed and automatic way (for example, to add some plots automatically to observers' end-of-night reports).

#### Catchup Service

This is a custom-written service which finds gaps in the processing for various channels, and runs those images through their respective "pipelines" (not `pipelineTask`s but the respective unit of work for a given instrument/image/channel). This is specifically used for catchup on snowflake services - full focal plane catchup processing a very differently shaped problem, and is covered in [Full Focal Plane Processing control](#processing-control).

#### ImageExaminer (service likely needs replacing)

For each new LATISS image, this service creates a `.png` containg some very quick (order of 1 second including i/o) canned analyses which are appropriate to run on all LATISS images, and sends it to S3 for display on RubinTV. This was a very quick and dirty service which was thrown together, and likely could do with a total rewrite now that other software has matured.

#### SpectrumExaminer

For each new dispersed (spectral) LATISS image taken, this service creates a `.png` containg a very quick (order of 1 second including i/o) "spectral reduction" and sends it to S3 for display on RubinTV. This provides realtime feedback for observers and scripts, giving the main source signal level and the continuum flux in ADU/s so that exposure times can be adjusted dynmaically.

#### AuxTel Mount Torque Analysis

For each LATISS image which is longer than 2s[^2s_limit] and for which the mount was moving, the mount position and motor currents (torques) are pulled for each of the three axes from the EFD. The residuals from nominal pointing are calulated, and the spurious mount motion RMS and the contribution this has to the image quality are calculated. This metric is sent to RubinTV, along with a plot of the mount motions. The mount performance metrics and image degradation numbers will be sent to Summit/Visit Database once that is possible.

[^2s_limit]: XXX Why is there a 2s limit on this? Need to get an explanation from Craig. I think it might be possible to fix this now.

#### TMA Event Generation

A `TMAEventMaker` is run in a loop, such that every time the TMA moves, new events are geneated from the EFD via the `TMAStateMachine` (see [SITCOMTN-098](https://sitcomtn-098.lsst.io/) for the technical details on `TMAEvent` generation). For each new TMA movement, various bits of telemetry are pulled from the EFD, and plots and metrics are created from them, and are sent to RubinTV for display (and will be sent to the Summit/Visit Database once it is possible).


### Full Focal Plane Processing

For the sake of simplicity, this section will only consider the summit, but note that simialar systems are in place at all locations, with no real architectural differences, just different `instruments`' data streams being processed.

For each `instrument` (which on the summit will be, at some point, likely all of `LATISS`, `LSSTComCam` and `LSSTCam` simultaneously, due to cleanroom operation), the architecture is as follows:

A head node exists, which watches for new data landing in the instrument's repo, and dispatches this in a dynamic and configurable way to the worker nodes, as detailed in [Processing Control](#processing-control).

Scatter/gather.
* ISR
* SFM

(processing-control)=
#### Processing control

Full focal plane processing behaviour is both dynamic and configurable: it is dynamic, based on the image acquisition rate, and how well the cluster is able keeping up. The behaviour for when we are not perfectly keeping up is configurable via LOVE (and notebooks for a _very_ select set of power users, and _only_ for development and testing use).

##### Dynamic focal plane selection

Work distribution via `redis`. Pods nominally have detector affinity, as this keeps things simple and allows caching of calibs, though there are plans to handle things more dynamically if it turns out this would allow making better use of resources. Fanout service allocates work based on the current `ProcessingMode`

* `As fast as possible`: each pod immediately

{images of focal plane patterns here}

##### Processing modes

* `WAITING`: Each detector's pod sits and waits for the next image to land. Once one is available, SFM runs to completion, `butler.put()`ing the resulting calexp as well as writing out some intermediate metadata shards which are collated and sent to Sasquatch and the Summit/Visit Database and RubinTV. Once the processing is finished, the pod waits and does not process any images in the backlog queue, waiting for a new image to land.
* `CONSUMING`: Each detector's pod processes its most recent detector image as per `WAITING`, but if, once finished, a new image has not landed, the most recent image on the backlog queue will be processed, continually, until a new image lands.
* `MURDEROUS`: This is a best-of-both-worlds hybrid of `WAITING` and `CONSUMING` where the most recent image is always let to run to completion, but if the image in progress came from the backlog queue and a new image lands while processing is not yet finished, progress is abandoned and the image is put back on the backlog queue. This mode does not yet exist.


#### Gather-type processes

`Plotter` and `RePlotter`. Usage of the scratch area.

## Notes

### `git-pull` on pod start

- Motivation
- How it works
    - list of packages
- responsibility.


### Techdebt

There are several places where Rapid Analysis has significant tech debt. Some of it is minor, but ugly and just in need of a tidyup because it was done in a hurry, but some of it is deeper and a risk to ongoing stability and maintainance, and hinders developement.

Minor:
* `LocationConfig`: This is honestly just pretty gross. It is not a hard fix, but it does need doing.
* Package (and some file) renaming. Need to finally deal with the hangover of `rubintv_production` etc.
* Some channels are just a bit ugly/hacky because they were written in a hurry and then ignored once they worked. They're only the low priority ones though, e.g. all sky camera.

Major:
* Test coverage - both unit tests and CI testing: The test coverage is frankly shocking - it's very close to zero (XXX measure this?). It not only makes development harder and more scary, but hinders larger, more refactoring-like work, and thus causes a vicious circle, with the debt compounding as the lack of tests makes tidying up more difficult itself.
* RA pod build system and devOps work: It may well be that there's no real tech debt here, it may just need more FTEs. Basically it just needs to be so that it doesn't need to be done me and Tiago (though I think Sebastian might be able to help here, now).


----------------

TODO:

* Install and run a spell checker
* Add diagrams
* Add section numbers - how do I do this automatically?
* Check for all instances of XXX
* Note to self: can I make an `ipywidget` to do an equivalent of the LOVE display of cluster status?
