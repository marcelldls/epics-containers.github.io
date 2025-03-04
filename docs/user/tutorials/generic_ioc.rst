Create a Generic IOC
====================

In this tutorial you will learn how to take an existing support module and
create a Generic IOC that builds it. You will also learn how to embed an
example IOC instance into the Generic IOC for testing and demonstration.

This is a type 2. change from the list at `ioc_change_types`.

Lakeshore 340 Temperature Controller
------------------------------------

The example we will use is a Lakeshore 340 temperature controller. This
is a Stream Device based support module that has historically been internal
to Diamond Light Source.

See details of the device:
`lakeshore 340 <https://www.lakeshore.com/products/categories/overview/discontinued-products/discontinued-products/model-340-cryogenic-temperature-controller>`_

.. note::

    DLS has an existing IOC building tool ``XML Builder`` for traditional
    IOCs. It has allowed DLS to a have concise way of describing a beamline for many
    years. However, it requires some changes to the support modules and for this
    reason DLS maintain's a fork of all upstream support modules it uses.
    epics-containers is intended to remove this barrier to collaboration and
    use support modules from public repositories wherever appropriate. This
    includes external publishing of previously internal support modules.

The first step was to publish the support module to a public repository,
it now lives at:

https://github.com/DiamondLightSource/lakeshore340

The project required a little genericizing as follows:

- add an Apache V2 LICENCE file in the root
- Make sure that configure/RELEASE has an include of RELEASE.local at the end
- change the make file to skip the ``XML Builder`` /etc folder

The commit where these changes were made is
`0ff410a3e1131 <https://github.com/DiamondLightSource/lakeshore340/commit/0ff410a3e1131c96078837424b2dfcdb4af2c356>`_

Something like these steps may be required when publishing any
facility's previously internal support modules.


Create a New Generic IOC project
--------------------------------

By convention Generic IOC projects are named ``ioc-XXX`` where ``XXX`` is the
name of the primary support module. So here we will be building
``ioc-lakeshore340``.

Much like creating a new beamline we have a template project that can be used
as the starting point for a new Generic IOC. Again we will create this in
your personal GitHub user space.

Go to the Generic IOC template project at:

https://github.com/epics-containers/ioc-template

Click on the ``Use this template`` button and create a new repository called
``ioc-lakeshore340`` in your personal GitHub account.

As soon as you do this the build in GitHub Actions CI will start building the
project. This will make a container image of the template project, but
not publish it because there is no release tag as yet. You can watch this
by clicking on the ``Actions`` tab in your new repository.

You might think building the template project was a waste of GitHub CPU. But,
this is not so, because of container build cacheing. The next time you build
the project in CI, with your changes, it will re-use most of the steps
and be much faster.

Prepare the New Repo for Development
------------------------------------

There are only three places where you need to change the Generic IOC template
to make your own Generic IOC.

#.  Dockerfile - add in the support modules you need
#.  README.md - change to describe your Generic IOC
#.  ibek-support - add new support module recipes into this submodule

To work on this project we will make a local developer container. All
changes and testing will be performed inside this developer container.

To get the developer container up and running:

.. code-block:: bash

    git clone git@github.com:<YOUR GITHUB ACCOUNT>/ioc-lakeshore340.git
    cd ioc-lakeshore340
    ./build
    code .
    # choose "Reopen in Container"

Once the developer container is running it is always instructive to have the
``/epics`` folder added to your workspace:

- File -> Add Folder to Workspace
- Select ``/epics``
- Click ignore if you see an error
- File -> Save Workspace As...
- Choose the default ``/workspaces/ioc-lakeshore340/ioc-lakeshore340.code-workspace``

Note that workspace files are not committed to git. They are specific to your
local development environment. Saving a workspace allows you to reopen the
same set of folders in the developer container, using the *Recent* list shown
when opening a new VSCode window.

Now is a good time to edit the README.md file and change it to describe your
Generic IOC as you see fit.

Initial Changes to the Dockerfile
---------------------------------

The Dockerfile is the recipe for building the container image. It is a set
of steps that get run inside a container. The starting container filesystem
state is determined by a ``FROM`` line at the top of the Dockerfile.

In the Generic IOC template the ``FROM`` line gets a version of the
epics-containers base image. It then demonstrates how to add a support module
to the container image. The ``iocStats`` support module is added and built
by the template. It is recommended to keep this module as the default
behaviour in Kubernetes is to use ``iocStats`` to monitor the health of
the IOC.

Thus you can start adding support modules by adding more ``COPY`` and ``RUN``
lines to the Dockerfile. Just like those for the ``iocStats`` module.

The rest of the Dockerfile is boilerplate and for best results you only need
to remove the comment below and replace it with the additional support
modules you need. Doing this means it is easy to adopt changes to the original
template Dockerfile in the future.

.. code-block:: dockerfile

    ################################################################################
    #  TODO - Add further support module installations here
    ################################################################################

Because lakeshore340 support is a StreamDevice we will need to add in the
required dependencies. These are ``asyn`` and ``StreamDevice``. We will
first install those inside our devcontainer as follows:

.. code-block:: bash

    # open a new terminal in VSCode (Terminal -> New Terminal)
    cd /workspaces/ioc-lakeshore340/ibek-support
    asyn/install.sh R4-42
    StreamDevice/install.sh 2.8.24

This pulls the two support modules from GitHub and builds them in our devcontainer.
Now any IOC instances we run in the devcontainer will be able to use these support
modules.

Next, make sure that the next build of our ``ioc-lakeshore340`` container
image will have the same support built in by updating the Dockerfile as follows:

.. code-block:: dockerfile

    COPY ibek-support/asyn/ asyn/
    RUN asyn/install.sh R4-42

    COPY ibek-support/StreamDevice/ StreamDevice/
    RUN StreamDevice/install.sh 2.8.24

The above commands added ``StreamDevice`` and its dependency ``asyn``.
For each support module
we copy it's ``ibek-support`` folder and then run the ``install.sh`` script. The
only argument to ``install.sh`` is the git tag for the version of the support
module required. ``ibek-support`` is a submodule used by all the Generic IOC
projects that contains recipes for building support modules, it will be covered
in more detail as we learn to add our own recipe for lakeshore340 below.

You may think that there is a lot of duplication here e.g. ``asyn`` appears
3 times. However, this is explicitly
done to make the build cache more efficient and speed up development.
For example we could copy everything out of the ibek-support directory
in a single command but then if I changed a StreamDevice ibek-support file the
build would have to re-fetch and re-make all the support modules. By
only copying the files we are about to use in the next step we can
massively increase the build cache hit rate.

.. note::

    These changes to the Dockerfile mean that if we were to exit the devcontainer,
    and then run ``./build`` again, it would would add the ``asyn`` and
    ``StreamDevice`` support modules to the container image. Re-launching the
    devcontainer would then have the new support modules available right away.

    This is a common pattern for working in these devcontainers. You can
    try out installing anything you need. Then once happy with it, add the
    commands to the Dockerfile, so that these changes become permanent.


Prepare The ibek-support Submodule
----------------------------------

Now we are ready to add the lakeshore340 support module to our project. In
order to do so we must first add a recipe for it to ``ibek-support``.

The ``ibek-support`` submodule is used to share information about how to build
and use support modules. It contains three kinds of files:

#.  install.sh - These are used to fetch and build support modules. They are
    run from the Dockerfile as described above.

#.  IBEK support module ``definitions``: These are used to help IOCs build their
    iocShell boot scripts and EPICS Database from YAML descriptions.

#.  PVI definitions: These are used to add structure to the set of PV's a
    device exposes. This structure allows us to auto-generate engineering
    screens for the device. See https://github.com/epics-containers/pvi.

``ibek-support`` is curated for security reasons, therefore we need to work with
a fork of it so we can add our own recipe for lakeshore340. If you make changes
to ``ibek-support`` that are generally useful you can use a pull request to get them
merged into the main repo.

Perform the following steps to create a fork and update the submodule:

- goto https://github.com/epics-containers/ibek-support/fork
- uncheck ``Copy the main branch only``
- click ``Create Fork``
- click on ``<> Code`` and copy the *HTTPS* URL
- cd to the ioc-lakeshore340 directory

.. code-block:: bash

    git submodule set-url ibek-support <PASTE *HTTPS* URL HERE>
    git submodule init
    git submodule update
    cd ibek-support
    git fetch
    git checkout tutorial-KEEP # see note below
    cd ..

We are using the ``tutorial-KEEP`` branch which is a snapshot of the ``ibek-support`` state
appropriate for this tutorial. Normally you would use the ``main`` branch and
then create your own branch off of that to work in.

.. note::

    IMPORTANT: we used an *HTTPS* URL for the ``ibek-support`` submodule, not
    a *SSH* URL. This is because other clones of ``ioc-lakeshore340`` will not
    be guaranteed to have the required SSH keys. HTTPS is fine for reading, but
    to write you need SSH. Therefore add the following to your ``~/.gitconfig``:

    .. code-block::

        [url "ssh://git@github.com/"]
                insteadOf = https://github.com/

    This tells git to use SSH for all GitHub URLs, when it sees an HTTP URL.


The git submodule allows us to share the ``ibek-support`` definitions between all
ioc-XXX projects but also allows each project to have its copy fixed to
a particular commit (until updated with ``git pull``) see
https://git-scm.com/book/en/v2/Git-Tools-Submodules for more information.


Create install.sh For The lakeshore340
--------------------------------------

The first file we will create is the ``install.sh`` script for lakeshore340.
This is a simple script that fetches the support module from GitHub and
builds it.

These scripts draw heavily on the ``ibek`` tool to do tasks that most support
modules require. They are also as close to identical as possible for simple
support modules.

IMPORTANT points to note are:

- Although we are using ``ibek`` we are really just automating what an EPICS
  engineer would do manually. This is very much using the vanilla EPICS build
  system that comes with EPICS base, along with the vanilla Make and Config files
  that come with each support module. These steps are:-

  - make sure we have the necessary system dependencies installed
  - fetch a version of the support module from GitHub
  - add a RELEASE.local to enable dependency resolution
  - optionally add CONFIG_SITE.local to apply settings for the build environment
  - run make to build the support module
  - take a note of the dbds and libs that we build so that we can use them
    to make our IOC instance later
- This is a bash script so although we encourage a very standard structure,
  you can do anything you like. For example this support module has to
  compile a 3rd party library before it can build the support module itself.
  `ADAravis install.sh <https://github.com/gilesknap/ibek-support/blob/46fd9394f6bf07da97ab7971e6b3f09a623a42f6/ADAravis/install.sh#L17-L44>`_

To make your lakeshore340 install.sh script:

.. code-block:: bash

    cd ibek-support
    mkdir lakeshore340
    cp iocStats/install.sh lakeshore340/install.sh
    code lakeshore340/install.sh

Now edit the install.sh script to look like the code block below.

The changes required for any support module you care to build would be:

- change the NAME variable to match the name of the support module
- add in ``ibek support apt-install`` lines for any system dependencies.
  These can be for the developer stage or the runtime stage or both.
- change the ``ibek support add-*`` lines to declare the libs and DBDs
  that this module will publish.
- add extra release macros for RELEASE.local (the RELEASE macro for
  the current support module is added automatically). Or add
  CONFIG entries for CONFIG_SITE.local as required.
  None of these were required for lakeshore340. To see how to use these
  functions see

  - ibek support add-release-macro --help
  - ibek support add-to-config-site --help

.. code-block:: bash

    #!/bin/bash

    # ARGUMENTS:
    #  $1 VERSION to install (must match repo tag)
    VERSION=${1}
    NAME=lakeshore340
    FOLDER=$(dirname $(readlink -f $0))

    # log output and abort on failure
    set -xe

    # doxygen is used in documentation build for the developer stage
    ibek support apt-install --only=dev doxygen

    # get the source and fix up the configure/RELEASE files
    ibek support git-clone ${NAME} ${VERSION} --org https://github.com/DiamondLightSource/

    ibek support register ${NAME}

    # declare the libs and DBDs that are required in ioc/iocApp/src/Makefile
    # None required for a stream device ------------------------------------
    #ibek support add-libs
    #ibek support add-dbds

    # compile the support module
    ibek support compile ${NAME}
    # prepare *.bob, *.pvi, *.ibek.support.yaml for access outside the container.
    ibek support generate-links ${FOLDER}

Having made these changes you can now test the script by running it:

.. code-block:: bash

    cd /workspaces/ioc-lakeshore340/ibek-support
    chmod +x lakeshore340/install.sh
    lakeshore340/install.sh 2-6-2

You now have lakeshore340 support in your developer container. Let's go ahead
and add that into the Dockerfile:

.. code-block:: dockerfile

    COPY ibek-support/lakeshore340/ lakeshore340/
    RUN lakeshore340/install.sh 2-6-2

This means you can compile an IOC with lakeshore340 support in this container
but we don't yet have a way to generate startup scripts and EPICS Databases
for the instances. We will do that next.

Create Support YAML for the lakeshore340
----------------------------------------

When making an IOC instance from a Generic IOC, the instance needs to supply
an iocShell startup script and an EPICS Database. You can supply hand
crafted ``st.cmd`` and ``ioc.subst`` files for this purpose. The Generic IOC
we have made above is already capable of using such files.

For this exercise we will use the full capabilities of ``ibek`` to generate
these files from a YAML description of the IOC instance. To do this we need
to create a YAML file that describes what the instance YAML is allowed to
make.

TODO: a detailed description of the YAML files' structure and purpose should
be included in the ``ibek`` documentation and linked here.
The current version of this is here
`entities <https://epics-containers.github.io/ibek/main/developer/explanations/entities.html>`_
but it is rather out of date.

To create an ``ibek`` support YAML file we need to provide a list of ``definitions`` .
Each ``definition`` gives:

- a name and description for the ``definition``
- a list of arguments that an
  instance of this ``definition`` may supply, with each having:

  - a type (string, integer, float, boolean, enum)
  - a name
  - a description
  - optionally a default value

- A list of database templates to instantiate for each instance of this ``definition``
  - including values for the Macros in the template

- A list of iocShell command line entries to add before or after ``iocInit``

In all of the fields Jinja templating can be used to combine the values of
arguments into the final output. At its simplest this is just the name of
an argument in double curly braces e.g. ``{{argument_name}}``. But, it can
also be used to do more complex things as a Python interpreter is evaluating
the text inside the curly braces and that interpreter has the values of
all the ``definition`` arguments available in its context.
See https://jinja.palletsprojects.com/en/3.0.x/templates/


.. note::

  IMPORTANT: the file created below MUST have the suffix ``.ibek.support.yaml``.
  This means it is a support yaml file for ``ibek``. This is important because
  when ``install.sh`` calls ``ibek support generate-links`` it will look for
  files with this suffix and make links to them in the ``ibek-defs`` folder.

  In turn when you run ``ibek ioc generate-schema`` it will look in the
  ``ibek-defs`` folder for all the support definition YAML files and combine
  them into a single schema file.

To make a lakeshore340 YAML file, go to the folder
``/workspaces/ioc-lakeshore340/ibek-support/lakeshore340/``
and create a file called ``lakeshore340.ibek.support.yaml``. Add the following
contents:

.. code-block:: yaml

  # yaml-language-server: $schema=https://github.com/epics-containers/ibek/releases/download/1.6.2/ibek.support.schema.json

  module: lakeshore340

  defs:
    - name: lakeshore340
      description: |-
        Lakeshore 340 Temperature Controller
        Notes: The temperatures in Kelvin are archived once every 10 secs.
      args:
        - type: str
          name: P
          description: |-
            Prefix for PV name

        - type: str
          name: PORT
          description: |-
            Bus/Port Address (eg. ASYN Port).

        - type: int
          name: ADDR
          description: |-
            Address on the bus
          default: 0

        - type: int
          name: SCAN
          description: |-
            SCAN rate for non-temperature/voltage parameters.
          default: 5

        - type: int
          name: TEMPSCAN
          description: |-
            SCAN rate for the temperature/voltage readings
          default: 5

        - type: id
          name: name
          description: |-
            Object and gui association name

        - type: int
          name: LOOP
          description: |-
            Which heater PID loop to control (Default = 1)
          default: 1

      databases:
        - file: $(LAKESHORE340)/db/lakeshore340.template
          args:
            name:
            SCAN:
            P:
            TEMPSCAN:
            PORT:
            LOOP:
            ADDR:

      pre_init:
        - value: |
            epicsEnvSet "STREAM_PROTOCOL_PATH", "$(LAKESHORE340)/lakeshore340App/protocol/"

This file declares a list of arguments, one for each of the database template
macros that it needs to substitute. It then declares that we need to instantiate
the ``lakeshore340.template`` database template and passes all of the arguments
verbatim to the template.

Finally it declares that we need to add a line to the iocShell startup script
that allows the IOC to find the module's StreamDevice protocol files.

Note that in the list of DB args or in the startup lines we can use combinations
of arguments to make the final output.

e.g. to make a more descriptive PV prefix we could use:

.. code-block:: yaml

      databases:
        - file: $(LAKESHORE340)/db/lakeshore340.template
          args:
            P: "{{P + ':' + name + ':'}}"

Finally, also note that the top line refers to a schema file. This is the global
``ibek`` schema for support module definition YAML. A single schema is used
for all support modules and is published along side the latest release of ``ibek``.
This means that a schema aware editor can provide auto-completion and validation
for your support module YAML files. The VSCode extension here
https://marketplace.visualstudio.com/items?itemName=redhat.vscode-yaml
adds this capability.

.. note::

    Because this is a DLS module originally, it has an ``etc/builder.py`` file
    that is used by the ``XML Builder`` tool. ``ibek`` has a converter
    that will translate this file into an ``ibek`` YAML file. Only DLS users
    can take advantage of this because it needs access to all the dependent
    DLS support module forks to work. See `../how-to/builder2ibek.support`

Example IOC instance
--------------------

In order to test our Generic IOC we now require an IOC instance to launch it.
For this exercise we will build an example instance right into the Generic IOC.
This is a great way to allow developers to experiment with the container,
but it is most likely to require a simulation of some kind to take the place
of a real piece of hardware for the instance to talk to.

Before creating the instance it is useful to have a schema for the YAML we
are about to write. To generate a schema for this specific Generic IOC
perform the following command:

.. code-block:: bash

    ibek ioc generate-schema  > /tmp/ibek.ioc.schema.json

This will make a schema that allows declaration of instances of the
definitions defined in the support YAML file we made above. But ALSO combines
in the definitions from the ``devIocStats`` support module and all other
modules that have been built inside this container.

Once this repository is published to GitHub, the schema will be available
as part of the release at the following URL:

.. code-block::

    https://github.com/<YOUR GITHUB ACCOUNT>/ioc-lakeshore340/releases/download/<VERSION TAG>/ibek.ioc.schema.json

This would then be the URL you would put at the top of any IOC instances using
your released Generic IOC.

To create the instance we create a folder:

    /workspaces/ioc-lakeshore340/ioc-examples/bl16i-ea-ioc-07/config/

and create a file in there called:

    bl16i-ea-ioc-07.yaml

with the following contents:

.. code-block:: yaml

  # yaml-language-server: $schema=/tmp/ibek.ioc.schema.json

  ioc_name: "{{ ioc_yaml_file_name }}"

  description: auto-generated by https://github.com/epics-containers/builder2ibek

  entities:
    - type: devIocStats.iocAdminSoft
      IOC: "{{ ioc_name | upper }}"

    - type: asyn.AsynIP
      name: p1
      port: 127.0.0.1:5401

    - type: lakeshore340.lakeshore340
      ADDR: 12
      LOOP: 2
      P: BL16I-EA-LS340-01
      PORT: p1
      SCAN: 5
      TEMPSCAN: 2
      name: lakeshore


The above YAML file declares an IOC instance that has the following 3
``entities`` (which is what we call instances of ``definitions`` in ``ibek``):

- A devIocStats object that will supply monitoring PVs
- An asyn IP port that will be used to talk to the simulator
- A lakeshore340 object that will talk to the simulator via the asyn port

This instance is now ready to run inside the developer container. To do so
perform the following steps:

.. code-block:: bash

    cd /epics/support/lakeshore340/etc/simulations/
    ./lakeshore340_sim.py

Now create a new terminal in VSCode (Terminal -> New Terminal) and run:

.. code-block:: bash

    ibek dev instance /workspaces/ioc-lakeshore340/ioc-examples/bl16i-ea-ioc-07
    cd /epics/ioc
    make
    ./start.sh

If all is well then you should see the IOC start up and connect to the
simulator. You will see the simulator logging the queries it receives.

TODO: it is possible to launch the bob file in:

    /epics/support/lakeshore340/lakeshore340App/opi/bob/lakeshore340.bob

to see a GUI for this IOC instance. However, I'm reserving writing about
GUI until I have the PVI integration done on this module and we can see
the auto-generated GUI.

To investigate what ``ibek`` did to make the Generic IOC binary and the
IOC instance files, take a look at the following files.

- ``/epics/runtime`` - the runtime assets created from a combination of the
    instance YAML and all the referenced support YAML

- ``/epics/ioc/iocApp/Makefile`` - this picks up the libs and DBDs from the
  support module builds which record their dbds and libs in:

  - ``/epics/support/configure/dbd_list``
  - ``/epics/support/configure/lib_list``

- ``/epics/ioc/support/configure/RELEASE`` - a global release file that contains
  macros for all the support built in the container. This is soft linked
  to ``configure/RELEASE.local`` in each support module.

- ``/epics/support/configure/RELEASE.shell`` - created along with the global
  release file. Sets all the release macros as shell environment variables
  for passing into the ioc startup script.

.. note::

    Because this IOC instance is a copy of a real IOC at DLS it comes
    from a builder XML file originally. DLS users with builder beamlines
    can use ``builder2ibek`` to convert their builder XML files into
    ``ibek`` YAML IOC instance files. See `../how-to/builder2ibek`.
    Note this is distinct from making support YAML files with
    ``builder2ibek.support``.

Experimenting With Changes to the IOC Instance and Generic IOC
--------------------------------------------------------------

Inside the developer container you can add and remove support, change the
IOC instance YAML file and re-build the IOC instance until everything is
working as you want it to. At that point you can push the changes to GitHub
and the CI should build a container image. Once that has succeeded you can
tag the release and the CI will publish the container image to GHCR.

Note that building the IOC binary is required after any change to the set
of support modules inside this container. However it is not required after
changes to the IOC instance YAML file. If you want to change the instance
you can:

- edit the YAML file
- stop the IOC
- start the IOC with ``./start.sh``
- that's it





