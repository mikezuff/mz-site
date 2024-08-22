---
title: "legacy code rewrite"
tags: ["golang"]
date: 2022-08-22
---

## Disclaimer:

This is a story about successfully rewriting some fairly simple legacy software.

Your legacy software is likely not as simple to rewrite.

**You should not rewrite your legacy software.**

## What is legacy software?

Legacy software is:
- something written long ago, maybe by someone else
- likely to be earning revenue right now
- written in some outdated programming language with the techniques of the time, without tests
- likely to have awkward parts added on over time, some done in haste under time pressure

## My Legacy Codebase

At my last employer there was a server health monitoring system called `healthd`.
`healthd` was a heap of inheritance-based C++ that had been in operation at the company for 17 years.
Its source code was intermingled with other code implementing another obsolete service.
It had a few quirks but did the job most of the time.

As part of a larger effort to decommission hundreds of aging physical FreeBSD servers, I needed to move the `healthd` service to different servers.
The available servers were running Ubuntu Linux.
I hoped it would be an easy task: create a [GitLab CI](https://docs.gitlab.com/ee/ci/) job to build and package the existing code for Ubuntu, then create a little Saltstack code to deploy it.

The port was easy, but testing found that some quirks seemed to be exacerbated and there was growth in memory usage suggestive of a leak.
On top of that, for years the company had been putting up with one intermittent failure mode that had recently caused another major issue.
This meant that there was finally some support for making changes to fix the failure mode.
Should I be the one to brave the legacy code change?

## The Rewrite

If I was going to fix these issues then I either had to re-learn how to debug/profile C++ and patch the issues or rewrite the system in Go.
I have decades of experience with legacy software.
That experience leads me to strongly prefer the former option.
Sometimes though, the best option is a fresh start, which proved to be the case for this project.

## Code Structure

This was the perfect application for Go.
I expected that a Go implementation would likely take very little time to write and would allow solving the behavior issues while also drastically improving maintainability.

`healthd` is a distributed server health monitoring system.
Instances are run within all datacenter and monitor servers within that datacenter.

![healthd system context](/images/healthd-context.drawio.png)

Clients in other datacenters connect to a local instance and request a `monitor` of a target local server.
One or more `monitors` is satisfied by a `check`, which periodically confirms the proper operation of the target.
These user-concepts map well to Go structs an goroutines make the code easy to write and to understand.
These comprise the architecture of `healthd` along with an additional component to manage the `client` socket and interface.

This diagram shows their relationship:

![component diagram](/images/healthd-components.drawio.png)

When a client connects, a new `client` goroutine is created listening for commands.
The monitor command creates a `monitor` which links the client's request to the one unique `check` which is created when a target is first monitored.
Each `check` is a goroutine that performs the monitoring, notifies the attached monitors, then sleeps for the required monitoring interval.


## Outcome

As a result of this rewrite I was able to resolve bugs, put the code under test, and simplify the codebase by implementing it in Go.

As measured by [cloc](https://github.com/AlDanial/cloc), the C++ code was almost 4000 lines of untested inheritance-based legacy.

```
[mikezuff@macbook legacy-healthd]$ cloc $LEGACY_FILES
      31 text files.
      31 unique files.
       0 files ignored.

github.com/AlDanial/cloc v 2.02  T=0.03 s (1167.7 files/s, 179480.4 lines/s)
-------------------------------------------------------------------------------
Language                     files          blank        comment           code
-------------------------------------------------------------------------------
C++                             19            578            229           3250
C/C++ Header                    12            150             26            532
-------------------------------------------------------------------------------
SUM:                            31            728            255           3782
-------------------------------------------------------------------------------
```

The Go code is 2300 lines of modern, tested, maintainable code.
Writing this code in a test-first style guided the implementation and resulted in solid test coverage.

```
[mikezuff@macbook go-healthd]$ cloc $GO_FILES
      20 text files.
      20 unique files.
       0 files ignored.

github.com/AlDanial/cloc v 2.02  T=0.03 s (589.3 files/s, 78557.4 lines/s)
-------------------------------------------------------------------------------
Language                     files          blank        comment           code
-------------------------------------------------------------------------------
Go                              16            297             93           2240
Text                             4              2              0             34
-------------------------------------------------------------------------------
SUM:                            20            299             93           2274
-------------------------------------------------------------------------------
```

I call that a win.
