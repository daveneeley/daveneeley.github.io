---
layout: post
title:  "Working for motorola solutions"
date:   2017-02-17 10:12:00
categories: work
---
Last November, my employer Spillman Technologies was acquired by Motorola Solutions. Not to be confused with Motorola Mobility, the maker of awesome cell-phones. I had the privilege of taking my first flight to Chicago in 15-odd years. A couple of co-workers and I joined the newly formed Site Reliability Engineering Team for cloud offerings, including folks from Poland, Pittsburgh, Chicago, and Salt Lake. An excellent group of people.

The time leading up to the acquisition (and the few months of uncertainty afterwards), were quite stressful for my team. We were asked to set up a new virtualization environment, from the ground up, with zero lead time and very little background information on why this was suddenly so important. About two weeks before we had flipped the switch on our Subversion to Git conversion. We did this on a release boundary, which was good and bad. It meant that the most stable code still existed, and was delivered daily, from Subversion. But it also meant that all the developers were excited to get going on their new projects for the next release.

This would have been fine, except that we didn't just clone the Subversion repository. The monolithic beast that it was, we filtered it into five Git repositories. That involved re-pathing all the installers and getting all the interdependencies resolved. We had a large amount of that work done before the rollout date, but not all. And we learned there were more interdependencies as we did further testing on the Git code. To make the matter more challenging, we also converted all of our Java-based code from manual ant-based builds to Maven. Once again, most of that work was done before the rollout, but not all. 

Why did we roll out when we everything wasn't done? An excellent question. We felt we absolutely had to hit the post-release delivery window with the best stuff we had, or wait four to five more months for the next window, keeping Subversion and Git in sync that whole time. Keeping the two in sync would have been fine, except for the switch to Maven. We felt we had to rollout Maven at the same time as Git because of one of our development practices (which was perfectly acceptable in the Subversion model): checked in jars in source control. This is a git no-no. Not because it can't handle it, but just because it's a *bad idea*. Every binary ever checked in gets included with every git clone. It's data-splosion. Our Java codebase is not small. At conversion time it included 118 individual Java projects, mostly webapps. The working copy size of the Java project was 2GB! To convert all of these to Maven required considerable effort, and keeping it in sync was a largely manual task, frought with **peril**!.

And last, but not least, one of the first features to implement in the first git-based release was a major upgrade to our database engine. So, to recap, all of the following was going on at work:
- Converted from Subversion to Git
- Introduced Maven
- Support 70-some developers on both new tools, both crucial to their workflow
- Still designing and implementing new git-based workflows to accompany the transition
- Get Maven release-ready
- Get product release-ready (including many previously defined Git task that were required for release but not rollout)
- Major database engine upgrade

Now the thing about this new high priority but low visibility project was that it didn't have anything to do with any of the above. It was on a completely different product line. Different everything. Our team began helping out on production support of this other product-line in production, but hadn't dealt with it much prior to that.
