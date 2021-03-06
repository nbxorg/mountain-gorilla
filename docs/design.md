---
title: MG Design Discussions
markdown2extras: tables, cuddled-lists
apisections:
---
<!--
    This Source Code Form is subject to the terms of the Mozilla Public
    License, v. 2.0. If a copy of the MPL was not distributed with this
    file, You can obtain one at http://mozilla.org/MPL/2.0/.
-->

<!--
    Copyright (c) 2014, Joyent, Inc.
-->

# MG Design Discussions

**This is a Joyent internal document.** Here-in some design discussions for
MG, collected here because they might be helpful to understand why MG
is the way it is. Each section is dated to give context if reading this
in the future when Amon design might have moved on. *Add new sections to
the top.*


# build and deploy with same dataset (19-Apr-2012)

Here is the problem: The MG build machine, e.g. the "master" Jenkins build
slave is a zone based on dataset smartos-1.3.18 (that's a guess, I'm not
sure which dataset it is). The "moray" component built on this build slave
is *deployed* to a zone based on dataset smartos-1.6.0. These have different
gcc's and different libstdc++. This has shown to result in, at least,
node 0.7.x builds blowing up.

Basically, given g++ compat for different versions being questionable and
v8 being heavy C++, a rule: **Thou shalt build on the same dataset as to
which you deploy.**

Note that while the rule talks about "dataset", likely the differentiator is
which *pkgsrc* version is being used. For the time being we'll be a little
more paranoid/convenient and talk in terms of scoping to the same dataset.
Also note that we aren't hitting crash problems currently with other software,
or with node **0.6.x** builds, so the hard commandment above could be
too strict.

This is a hard change from the current state of building all of the SDC world
on one machine (current `./configure && make`). Two suggested ways to
attempt to add support for this (other suggestions welcome):

1.  Jenkins controlled. Having the *Jenkins Job configuration* for each
    component, e.g. moray here, specify the build slave on which it must
    build. Add a smartos-1.6.0-based build slave.

2.  MG controlled. Make MG all fancy-ass 'n shit: I.e. it manages the whole
    build and knows to task out building sub-components to separate zones
    (e.g. moray on a smartos-1.6.0 zone). Possibly MG would handle
    provisioning that zone itself if necessary. MG would then gather the
    results and put everything together.


Jenkins controlled pros and cons:

- Pro: Easier to get going. Create a Jenkins slave for each deployment
  dataset and tie that job's build to that build slave.
- Con (maybe Pro): Requires stopping using the full MG build (the current
  "sdc" Jenkins job) which builds the world on one machine. The "usbheadnode"
  Jenkins Job which just pulls all prebuilt pieces together, because the
  primary Jenkins producer of SDC release bits. Is this actually a Con? It is
  one less Jenkins Job that builds the output bits to worry about.
- Con: The knowledge of build requirements (on which dataset to build) is
  spread into Jenkins. Frankly this isn't a huge loss because the relevant
  dataset info is currently *already* spread between the particular repo,
  e.g. moray.git, and "usb-headnode.git/zones/moray/setup.json".

MG controlled pros and cons:

- Con: Having MG control tasking out build steps to separate zones, and
  possibly having it dynamically provision those zones, is a lot of work.
  It would require a working SDC headnode -- presuming we'd use an SDC for
  provisioning rather than GZ access and manual "vmadm create" usage.
- Pro: That headnode setup for our build system would be great dogfood. :)
  Possibly even include compute nodes to spread out the build load.


My (Trent's) opinion right now is to just go with "Jenkins controlled".
Implementation details:

-   Add build slaves for each required dataset:

        $ cat usb-headnode/zones/*/dataset | sort | uniq
        smartos-1.3.18
        smartos-1.6.0
        smartos64-1.4.7
        smartos64-1.6.0

-   Add safeguards in each project to check that they are building on the
    dataset they expect. Thinking of adding the dataset name to the build
    output filename. E.g. we'd have:

        moray-pkg-master-20120419T155730Z-g22f7c32-smartos-1.6.0.tar.bz2

    An alternative would be for each component build to publish a separate
    build config data file, so you'd have:

        https://stuff.joyent.us/stuff/builds/moray/master-latest/moray/build-config.json

    with relevant binary compat build info.

    The safeguard is that usbheadnode build will bork if the dataset in the
    package filename doesn't match the "zones/ZONENAME/dataset" value.

    Not sure the best way to get that dataset name from within the build
    zone. Anybody? Could be passed in from Jenkins in the build environment.

IOW, pretty straightforward. Thoughts?

