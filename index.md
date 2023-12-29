# The Rapid Analysis Framework and its relationship to RubinTV

> This technote is both a work-in-progress, because
> * 1) it is a provisional version which won't be tidied up until after the design review
> * 2) it is documenting a moving target, meaning that there are places where the present tense is used to save having to write everything twice, but where the work hasn't quite been done yet. These are, however, all places where the work is planned and underway, _i.e._ it's not aspirational, it's just not quite finsihed yet, but will be on the scale of weeks.

## Abstract

```{abstract}
Documentation on the architecture of the Rapid Analysis Framework, including its relationship to RubinTV.
```

## Introduction

Historically, there has been confusion about what Rapid Analysis is, what RubinTV is, and what the relationship and boundaries are between the two. This is because they were developed at the same time, co-evolving with one another. However, they are entirely decoupled. Therefore, we begin with definitions of each, paying careful attention to say exactly what each is, and is not. This technote will provide a detailed description of, and motivation for the Rapid Analysis Framework, but does not go into depth about the workins of RubinTV - that will be covered in enough depth to make it clear what it is and where the boundaries are, but no more than that. It is also worth nothing that some of this confusion came from some poor naming choices as the co-evolution of Rapid Analysis and RubinTV was ongoing, almost all of which has now been rectified, with the only item remaining to be fixed being that the RA framework still lives in a repo called `rubintv_production`

#### What Rapid Analysis is/isn't

Rapid Analysis (RA) is a realtime processing framework. It is a set of kubernetes pods which run in various locations, and process data in realtime. Currently, these locations are:
* On the summit
* USDF
* BTS
* TTS

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

### Snowflake Services

### Full Focal Plane Processing

For the sake of simplicity, this section will only consider the summit, but note that simialar systems are in place at all locations, with no real architectural differences, just different `instruments` and data streams being processed.

For each `instrument` (which on the summit will be, at some point, likely all of `LATISS`, `LSSTComCam` and `LSSTCam` simultaneously), the architecture is as follows.

A head node exists, which watches for new data landing, and dispatches this to the worker nodes, in a scatter-type processing flow.

Scatter/gather. Processing control
* ISR
* SFM

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

