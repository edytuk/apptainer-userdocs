.. _definition-files:

################
Definition Files
################

.. _sec:deffiles:

{AProject} Definition File (or "def file" for short) is like a set
of blueprints explaining how to build a custom container. It includes
specifics about the base OS to build or the base container to start
from, as well as software to install, environment variables to set at
runtime, files to add from the host system, and container metadata.

********
Overview
********

{AProject} Definition file is divided into two parts:

#. **Header**: The Header describes the core operating system to build
   within the container. Here you will configure the base operating
   system features needed within the container. You can specify the
   Linux distribution, the specific version, and the packages that must
   be part of the core install (borrowed from the host system).

#. **Sections**: The rest of the definition is comprised of sections,
   (sometimes called scriptlets or blobs of data). Each section is
   defined by a ``%`` character followed by the name of the particular
   section. All sections are optional, and a def file may contain more
   than one instance of a given section. Sections that are executed at
   build time are executed with the ``/bin/sh`` interpreter and can
   accept ``/bin/sh`` options. Similarly, sections that produce scripts
   to be executed at runtime can accept options intended for
   ``/bin/sh``.

For more in-depth and practical examples of def files, see the `{Project}
examples repository <https://github.com/{orgrepo}/tree/{repobranch}/examples>`_.

For a direct comparison between Dockerfiles and {Project} definition
files, you can jump directly to :ref:`the relevant section
<sec:deffile-vs-dockerfile>` in the documentation.

******
Header
******

The header should be located at the beginning of the def file. It tells
{Project} about the base operating system that it should use to
build the container. It is composed of several keywords.

The only keyword that is required for every type of build is
``Bootstrap``. It determines the *bootstrap agent* that will be used to
create the base operating system you want to use. For example, the
``docker`` bootstrap agent will pull docker layers from `Docker Hub
<https://hub.docker.com/>`_ as a base OS from which to start your image.

The ``Bootstrap`` keyword needs to be
the first entry in the header section. This breaks compatibility with
older versions, which allowed the parameters of the header to appear in
any order.

Depending on the value assigned to ``Bootstrap``, other keywords may
also be valid in the header. For example, when using the ``docker``
bootstrap agent, the ``From`` keyword becomes valid. Here is an example
that uses the ``From`` keyword to build a Debian container:

.. code:: {command}

   Bootstrap: docker
   From: debian:7

A def file that uses an official mirror to install CentOS 7 might look
like this:

.. code:: {command}

   Bootstrap: yum
   OSVersion: 7
   MirrorURL: http://mirror.centos.org/centos-%{OSVERSION}/%{OSVERSION}/os/$basearch/
   Include: yum

Each bootstrap agent enables its own options and keywords, which you can
read about, and see examples of, in the :ref:`appendix <buildmodules>`.

Preferred bootstrap agents
==========================

-  :ref:`docker <build-docker-module>` (images hosted on Docker Hub)
-  :ref:`oras <build-oras>` (images from supported OCI registries)
-  :ref:`localimage <build-localimage>` (images saved on your machine)
-  :ref:`scratch <scratch-agent>` (a flexible option for building a
   container from scratch)

Other bootstrap agents
======================

-  :ref:`library <build-library-module>` (images hosted on Library API Registries)
-  :ref:`shub <build-shub>` (images hosted on Singularity Hub)
-  :ref:`yum <build-yum>` (yum-based systems such as CentOS and
   Scientific Linux)
-  :ref:`debootstrap <build-debootstrap>` (apt-based systems such as
   Debian and Ubuntu)
-  oci (bundle compliant with OCI Image Specification)
-  oci-archive (tar files obeying the OCI Image Layout Specification)
-  :ref:`docker-daemon <docker-daemon>` (images managed by the
   locally running docker daemon)
-  :ref:`docker-archive <docker-archive>` (saved docker
   images)
-  :ref:`arch <build-arch>` (Arch Linux)
-  :ref:`busybox <build-busybox>` (BusyBox)
-  :ref:`zypper <build-zypper>` (zypper-based systems such as SUSE and
   openSUSE)

SIF Image Verification / Fingerprints Header
============================================

If the bootstrap image is in the SIF format, then verification will be
performed at build time. This verification checks whether the image has
been signed. If it has been signed, the integrity of the image is
checked, and the signatures matched against public keys if available.
This process is equivalent to running ``{command} verify`` on the
bootstrap image.

By default, a failed verification (e.g. against an unsigned image, or
one that has been modified after signing) will produce a warning, but
the build will continue.

To make it a requirement that the bootstrap image verifies correctly and
has been signed by one or more keys, you can use the ``Fingerprints:``
header.

.. code:: {command}

   Bootstrap: localimage
   From: test.sif
   Fingerprints: 12045C8C0B1004D058DE4BEDA20C27EE7FF7BA84,22045C8C0B1004D058DE4BEDA20C27EE7FF7BA84

If, at build time, the image is not signed with keys corresponding to
*all* of the listed fingerprints, the build will fail.

The ``Fingerprints:`` header can be used with bootstrap agents that
provide a SIF image. The ``library`` agent always retrieves a SIF image.
The ``localimage`` agent *can* be used to refer to SIF images, which
will work correctly with the ``Fingerprints:`` header, but also to other
types of images, which will not.

The ``Fingerprints:`` header has no effect if the bootstrap image is not
in SIF format.

.. note::

   The verification occurs before the bootstrap image is extracted into
   a temporary directory for the build process. The fingerprint check
   ensures the correct image was retrieved for the build, but does not
   protect against malicious changes that could be made during the build
   process on an already-compromised machine.

********
Sections
********

The main content of the bootstrap file is broken into sections.
Different sections add different content, or execute commands at
different times during the build process. Note that if any command
fails, the build process will halt.

Here is an example definition file that uses every available section. We
will discuss each section in turn, below. It is not necessary to include
every section (or any sections at all) within a def file. Furthermore,
multiple sections of the same name can be included and will be appended
to one another during the build process.

.. code:: {command}

   Bootstrap: docker
   From: ubuntu:22.04
   Stage: build

   %setup
       touch /file1
       touch ${{ENVPREFIX}_ROOTFS}/file2

   %files
       /file1
       /file1 /opt

   %environment
       export LISTEN_PORT=54321
       export LC_ALL=C

   %post
       apt-get update && apt-get install -y netcat
       NOW=`date`
       echo "export NOW=\"${NOW}\"" >> ${ENVPREFIX}_ENVIRONMENT

   %runscript
       echo "Container was created $NOW"
       echo "Arguments received: $*"
       exec echo "$@"

   %startscript
       nc -lp $LISTEN_PORT

   %test
       grep -q NAME=\"Ubuntu\" /etc/os-release
       if [ $? -eq 0 ]; then
           echo "Container base is Ubuntu as expected."
       else
           echo "Container base is not Ubuntu."
           exit 1
       fi

   %labels
       Author alice
       Version v0.0.1

   %help
       This is a demo container used to illustrate a def file that uses all
       supported sections.

Although the order of the sections in the def file is unimportant, they
have been documented below in the order of their execution during the
build process for ease of understanding.

%setup
======

During the build process, commands in the ``%setup`` section are first
executed on the host system *outside* of the container, after the base
OS has been installed. You can reference the container file system with
the ``${ENVPREFIX}_ROOTFS`` environment variable in the ``%setup``
section.

.. note::

   Be careful with the ``%setup`` section! This scriptlet is executed
   outside of the container on the host system itself.
   Commands in ``%setup`` can alter and potentially damage the host.

   Moreover, whether the code in ``%setup`` runs successfully and
   correctly will depend on the configuration of the host system. That
   is exactly the kind of environment-dependency that containerization
   is meant to circumvent, in the first place.

Consider the example from the definition file above:

.. code:: {command}

   %setup
       touch /file1
       touch ${{ENVPREFIX}_ROOTFS}/file2

Here, ``file1`` is created at the root of the file system **on the
host**. We'll use ``file1`` to demonstrate the usage of the ``%files``
section below. ``file2``, on the other hand, is created at the root of
the file system **within the container**.

More recent versions of {Project} provide the ``%files`` section,
which is a safer alternative to copying files from the host system into
the container during the build process.

%files
======

The ``%files`` section allows you to copy files into the container with
greater safety than using the ``%setup`` section. Its general form is:

.. code:: {command}

   %files [from <stage>]
       <source> [<destination>]
       ...

Each line is a ``<source>`` and ``<destination>`` pair. The ``<source>``
is either:

  #. A valid path on your host system
  #. A valid path in a previous stage of the build

while the ``<destination>`` is always a path into the current container. If the
``<destination>`` path is omitted it will be assumed to be the same as
``<source>``. To show how copying from the host system works, let's
consider the example from the definition file above:

.. code:: {command}

   %files
       /file1
       /file1 /opt

``file1`` was created in the root of the host file system during the ``%setup``
section (see above).  The ``%files`` scriptlet will copy ``file1`` from the root
of the host filesystem to the root of the container filesystem, and then make a
second copy of ``file1`` inside ``/opt`` within the container filesystem
(i.e., at ``/opt/file1``).

Files can also be copied from other stages by providing the source location in the
previous stage and the destination in the current container.

.. code:: {command}

   %files from stage_name
     /root/hello /bin/hello

The only difference in behavior between copying files from your host
system and copying them from previous build stages is that in the former
case, symbolic links are *followed*, while in the latter case, symbolic
links are *preserved as symbolic links*.

Files in the ``%files`` section are always copied before the ``%post``
section is executed, so that they are available during the build and
configuration process.

%app*
=====

In some circumstances, it may be redundant to build different containers
for each app with nearly equivalent dependencies. {Project} supports
installing apps within internal modules based on the concept of the
`Scientific Filesystem (SCIF) <https://sci-f.github.io/>`_. More
information on defining and using SCIF Apps can be found :ref:`here
<apps>`.

%post
=====

This section is where you can download files from the internet with
tools like ``git`` and ``wget``, install new software and libraries,
write configuration files, create new directories, etc.

Consider the ``%post`` section from the example definition file above:

.. code:: {command}

   %post
       apt-get update && apt-get install -y netcat
       NOW=`date`
       echo "export NOW=\"${NOW}\"" >> ${ENVPREFIX}_ENVIRONMENT

This ``%post`` scriptlet uses the Ubuntu package manager ``apt`` to
update the container and install the program ``netcat`` (that will be
used in the ``%startscript`` section below).

The script also sets an environment variable at build time. Note that
the value of this variable cannot be anticipated, and therefore cannot
be set earlier in the ``%environment`` section. For situations like
this, the ``${ENVPREFIX}_ENVIRONMENT`` variable is provided. Assigning a
value to this variable will cause it to be written to a file called
``/.singularity.d/env/91-environment.sh`` that will be sourced by the
container at runtime.

.. note::

   Variables set in the ``%post`` section through
   ``${ENVPREFIX}_ENVIRONMENT`` take precedence over those added via
   ``%environment``.

.. _def-test-section:

%test
=====

The ``%test`` section runs at the very end of the build process, and can
be used to validate the container using methods of your choosing. You
can also execute this scriptlet through the container itself, using the
``test`` command.

Consider the ``%test`` section from the example definition file above:

.. code:: {command}

   %test
       grep -q NAME=\"Ubuntu\" /etc/os-release
       if [ $? -eq 0 ]; then
           echo "Container base is Ubuntu as expected."
       else
           echo "Container base is not Ubuntu."
           exit 1
       fi

This (somewhat trivial) script tests whether the base OS is Ubuntu. You
can use the ``%test`` section to test whether binaries were
appropriately downloaded and built, or whether software works as
expected on custom hardware. If you want to build a container without
running the ``%test`` section (for example, if your build system does
not have the same hardware that will be used in your production
environment), you can do so by passing the ``--notest`` flag to the
build command:

.. code::

   $ {command} build --notest my_container.sif my_container.def

Running the test command on a container built with this def file yields
the following:

.. code::

   $ {command} test my_container.sif
   Container base is Ubuntu as expected.

One common use of the ``%test`` section is to run a quick check that the
programs you installed in the container are indeed present.

Suppose you've installed the program ``samtools``, by adding it to the
list of packages passed to ``apt-get install`` in the ``%post`` section:

.. code:: {command}

   %post
       apt-get update && apt-get install -y netcat samtools
       NOW=`date` echo "export NOW=\"${NOW}\"" >>
       ${ENVPREFIX}__ENVIRONMENT

``samtools`` prints a usage message when run without any options, so you
might decide to test that it can be run by writing the following in the
``%test`` section:

.. code:: {command}

   %test
       echo 'Looking for samtools...'
       samtools

If ``samtools`` is not successfully installed in the container, then
``{command} build`` (if run without the ``--notest`` flag) will
produce an error (such as ``samtools: not found``) during the test phase
of the build, and running ``{command} test`` will produce the same
error.

The problem with this approach is that, like many other programs,
``samtools`` returns a non-zero error code when run without its
mandatory options. So, while the ``%test`` section we just wrote will
print the usage message of ``samtools`` if ``samtools`` has been
installed, it will also report the error code (reflecting the absence of
mandatory options to ``samtools``), which is probably not what we want
in this case.

A better approach would therefore be to run ``samtools`` with the
``version`` option, and check that the output is what we expected. Here,
we do this by running ``grep`` on the output and checking that the
version number begins with "1":

.. code:: {command}

   %test
       echo 'Looking for samtools...'
       ( samtools --version | grep -q 'samtools 1' ) && echo 'Success!'

Because the ``%test`` section is a shell scriptlet, complex tests are
possible. Remember that your scriptlet should be written so it exits
with a non-zero error code if the test encounters a problem.

The ``%test`` scriptlet will run under ``sh`` or ``bash`` by default. You can
change the shell or interpreter that the test runs under by using a custom
hashbang (``#!``) as the first line in your ``%test`` section:

.. code:: {command}

   %test
      #!/bin/zsh

      echo "$(readlink /proc/$$/exe) is our shell"


In the ``%test`` section above, the ``#!/bin/zsh`` means that the test
code will be run by the zsh shell installed at ``/bin/zsh``. The
``echo`` statement given above will display the shell that is running
the script, confirming that this works.

A custom hashbang runs the specified shell from the container
filesystem, not the host. Therefore, ``zsh`` must be installed in the
*container*, and since ``zsh`` is not built into the base Ubuntu image,
it would have to be installed as part of the ``%post`` section for this
``%test`` code to work properly.

%environment
============

The ``%environment`` section allows you to define environment variables
that will be set at runtime. Note that these variables are made
available in the container at runtime, but not at build time. This means
that if you need the same variables during the build process, you should
also define them in your ``%post`` section. Specifically:

-  **during build**: The ``%environment`` section is written to a
   dedicated file in the container metadata directory. This file is not
   sourced.
-  **during runtime**: The file in the container metadata directory is
   sourced.

You should use the same conventions that you would use in a ``.bashrc``
or ``.profile`` file. Consider the ``%environment`` section from the
example definition file above:

.. code:: {command}

   %environment
       export LISTEN_PORT=54321
       export LC_ALL=C

The ``$LISTEN_PORT`` variable will be used in the ``%startscript``
section of the same example, discussed below. The ``$LC_ALL`` variable
is useful for many programs (especially those written in Perl) that
expect a locale to be set.

After building this container, you can use a command like the following
one to verify that the environment variables have been set appropriately
at runtime:

.. code::

   $ {command} exec my_container.sif env | grep -E 'LISTEN_PORT|LC_ALL'
   LISTEN_PORT=54321
   LC_ALL=C

To set a default value for a variable in the ``%environment`` section,
but adopt the value of a host environment variable if it is set, use
the following syntax:

.. code:: {command}

    %environment
       FOO=${FOO:-'default'}

The value of ``FOO`` in the container will take the value of ``FOO`` on
the host, or ``default`` if ``FOO`` is not set on the host or if
``--cleanenv`` / ``--containall`` have been specified.

.. note::

   Variables added to the ``${ENVPREFIX}_ENVIRONMENT`` file in the
   ``%post`` section will take precedence over variables set in the
   ``%environment`` section.

See :ref:`Environment and Metadata <environment-and-metadata>` for more
information about the {Project} container environment.

.. _startscript:

%startscript
============

Similar to the ``%runscript`` section, the contents of the
``%startscript`` section are written to a dedicated file within the
container at build time. This file is executed when the ``instance
start`` command is issued.

Consider the ``%startscript`` section from the example definition file
above:

.. code:: {command}

   %startscript
       nc -lp $LISTEN_PORT

Here, the netcat (``nc``) program is used to listen for TCP traffic on
the port indicated by the ``$LISTEN_PORT`` variable (set in the
``%environment`` section, above). The script can be invoked as follows:

.. code::

   $ {command} instance start my_container.sif instance1
   INFO:    instance started successfully

   $ netstat -ln | grep 54321
   tcp        0      0 0.0.0.0:54321           0.0.0.0:*               LISTEN

   $ {command} instance stop instance1
   Stopping instance1 instance of /home/vagrant/my_container.sif (PID=19035)

.. _runscript:

%runscript
==========

The contents of the ``%runscript`` section are written to a dedicated
file within the container that is executed when the container image is
run (either via the ``{command} run`` command or by :ref:`executing
the container directly <runcontainer>` as a command). When the container
is invoked, arguments following the container name are passed to the
runscript. This means that you can (and should) process arguments within
your runscript.

Consider the ``%runscript`` section from the example definition file
above:

.. code:: {command}

   %runscript
       echo "Container was created $NOW"
       echo "Arguments received: $*"
       exec echo "$@"

In this runscript, the time that the container was created is echoed via
the ``$NOW`` variable (set in the ``%post`` section, above). The options
passed to the container at runtime are printed as a single string
(``$*``) and then they are passed to echo via a quoted array (``$@``)
which ensures that all of the arguments are properly parsed by the
executed command. The ``exec`` preceding the final ``echo`` command
replaces the current entry in the process table (which originally was
the call to {Project}). Thus, the runscript shell process ceases to
exist, and only the process running within the container remains.

Running the container built using this def file will yield the
following:

.. code::

   $ ./my_container.sif
   Container was created Thu Dec  6 20:01:56 UTC 2018
   Arguments received:

   $ ./my_container.sif this that and the other
   Container was created Thu Dec  6 20:01:56 UTC 2018
   Arguments received: this that and the other
   this that and the other

The ``%runscript`` scriptlet will run under ``sh`` or ``bash`` by default. You
can change the shell or interpreter that the test runs under by using a custom
hashbang (``#!``) as the first line in your ``%runscript`` section:

.. code:: {command}

   %runscript
      #!/bin/zsh

      echo "$(readlink /proc/$$/exe) is our shell"


Just like in the |def-test-section|_, the ``#!/bin/zsh`` means that the
runscript code will be run by the zsh shell installed at ``/bin/zsh``.
The ``echo`` statement given above will display the shell that is
running the script, confirming that this works.

And just like in the |def-test-section|_, a custom hashbang runs the
specified shell from the container filesystem, not the host. Therefore,
``zsh`` must be installed in the *container*, and since ``zsh`` is not
built into the base Ubuntu image, it would have to be installed as part
of the ``%post`` section for this ``%runscript`` code to work properly.

.. |def-test-section| replace:: ``%test`` section

%labels
=======

The ``%labels`` section is used to add metadata to the file
``/.singularity.d/labels.json`` within your container. The general
format is a name-value pair.

Consider the ``%labels`` section from the example definition file above:

.. code:: {command}

   %labels
       Author d@sylabs.io
       Version v0.0.1
       MyLabel Hello World

Note that labels are key-value pairs. To define a new label, add a new
line of text to the ``%labels`` section. The portion of text up to the
first space will be taken as the label's name, and the portion following
it will be taken as the label's value.

In the previous example, the first label name is ``Author``` with a
value of ``alice``. The second label name is ``Version`` with a
value of ``v0.0.1``. Finally, the third label name is ``MyLabel`` with a
value of ``Hello World``.

You can inspect the available labels on your image by running the
following command:

.. code::

   $ {command} inspect my_container.sif

   Author: alice
   Version: v0.0.1
   MyLabel: Hello World
   org.label-schema.build-arch: amd64
   org.label-schema.build-date: Tuesday_1_March_2022_16:49:5_PST
   org.label-schema.schema-version: 1.0
   org.label-schema.usage: /.singularity.d/runscript.help
   org.label-schema.usage.apptainer.runscript.help: /.singularity.d/runscript.help
   org.label-schema.usage.apptainer.version: 1.0.0
   org.label-schema.usage.singularity.deffile.bootstrap: docker
   org.label-schema.usage.singularity.deffile.from: ubuntu:18.04
   org.label-schema.usage.singularity.deffile.stage: build

As you can see from this output, some labels are generated automatically
from the build process. You can read more about labels and metadata
:ref:`here <environment-and-metadata>`.

%help
=====

Any text in the ``%help`` section is transcribed into a dedicated
metadata file in the container during the build process. This text can
then be displayed using the ``run-help`` command.

Consider the ``%help`` section from the example definition file above:

.. code:: {command}

   %help
       This is a demo container used to illustrate a def file that uses all
       supported sections.

After building the help can be displayed like so:

.. code::

   $ {command} run-help my_container.sif
       This is a demo container used to illustrate a def file that uses all
       supported sections.

******************
Multi-Stage Builds
******************

Multi-stage builds are supported, where one
environment can be used for compilation, and the resulting binary can
then be copied into a different final environment. One of the important
advantages of this approach is that it allows for a slimmer final image
that does not require the entire development stack.

.. code:: {command}

   Bootstrap: docker
   From: golang:1.12.3-alpine3.9
   Stage: devel

   %post
     # prep environment
     export PATH="/go/bin:/usr/local/go/bin:$PATH"
     export HOME="/root"
     cd /root

     # insert source code, could also be copied from the host with %files
     cat << EOF > hello.go
     package main
     import "fmt"

     func main() {
       fmt.Printf("Hello World!\n")
     }
   EOF

     go build -o hello hello.go


   # Install binary into the final image
   Bootstrap: library
   From: alpine:3.9
   Stage: final

   # install binary from stage one
   %files from devel
     /root/hello /bin/hello

   %runscript
     /bin/hello

The names of stages (assigned using the ``Stage`` keyword) are
arbitrary. Each of these sections will be executed in the same order as
described for a single stage build, except that the files from the
previous stage are copied before the ``%setup`` section of the next
stage is carried out. Files can only be copied from stages declared
before the current stage in the definition. E.g., the ``devel`` stage in
the above definition cannot copy files from the ``final`` stage, but the
``final`` stage can copy files from the ``devel`` stage.

.. _apps:

*********
SCIF Apps
*********

`SCIF <https://sci-f.github.io/>`__ is a standard for encapsulating
multiple apps into a container. A container with SCIF apps has multiple
entry points, and it is easy to choose which one you want to run. Each
entry point can carry out a different task, with its own environment,
metadata, etc., without the need for a collection of different
containers.

{Project} implements SCIF, and you can read more about how to use it
below.

SCIF is not specific to {Project}. To learn more, take a look at the
project's site at https://sci-f.github.io/, which includes extended
tutorials, a detailed specification of the SCIF standard, and other
information.

SCIF %app* sections
===================

SCIF apps within {aProject} container are created using ``%app*``
sections in a definition file. These ``%app*`` sections, which will
impact the way the container runs a specific ``--app``, can exist
alongside any of the primary sections (i.e. ``%post``,``%runscript``,
``%environment``, etc.). As with other sections, the ordering of the
``%app*`` sections isn't important.

The following runscript demonstrates how to build 2 different apps into
the same container using SCIF modules:

.. code:: {command}

   Bootstrap: docker
   From: ubuntu

   %environment
       GLOBAL=variables
       AVAILABLE="to all apps"

   ##############################
   # foo
   ##############################

   %apprun foo
       exec echo "RUNNING FOO"

   %applabels foo
       BESTAPP FOO

   %appinstall foo
       touch foo.exec

   %appenv foo
       SOFTWARE=foo
       export SOFTWARE

   %apphelp foo
       This is the help for foo.

   ##############################
   # bar
   ##############################

   %apphelp bar
       This is the help for bar.

   %applabels bar
       BESTAPP BAR

   %appinstall bar
       touch bar.exec

   %apprun bar
       exec echo "RUNNING BAR"

   %appenv bar
       SOFTWARE=bar
       export SOFTWARE

An ``%appenv`` section is the app-specific equivalent of ``%environment``.

Similarly, ``%appinstall`` is like ``%post`` but for a particular app. Note that
just like the general ``%post`` section, ``%appinstall`` sections run at build
time. Thus, when building a container from a definition file containing
``%appinstall`` sections, the content of all of these sections will be
executed, even if later on the user ends up running only some of the apps
defined in the file and not others. This is why the
`SCIF Standard <https://sci-f.github.io/specification>`__ indicates that files &
directories that are app-specific, and are potentially mutually-exclusive with
the files & directories of other apps, be placed under the app-specific
``/scif/apps/<app-name>`` directory to avoid conflicts between different apps.

Installing apps into modules using the ``%app*`` sections enables the
``--app`` option, allowing commands like the following:

.. code::

   % {command} run --app foo my_container.sif
   RUNNING FOO

This runs a specific app, ``foo``, from the multi-app container we
built.

The same environment variable, ``$SOFTWARE`` is defined for both apps in
the def file above. You can execute the following command to search the
list of active environment variables and ``grep`` to determine if the
variable changes depending on the app we specify:

.. code::

   $ {command} exec --app foo my_container.sif env | grep SOFTWARE
   SOFTWARE=foo

   $ {command} exec --app bar my_container.sif env | grep SOFTWARE
   SOFTWARE=bar

*******************************************
Best Practices for Writing Definition Files
*******************************************

When crafting your definition file, it is best to consider the
following:

#. Always install packages, programs, data, and files into operating
   system locations (e.g. not ``/home``, ``/tmp`` , or any other
   directories that might get commonly bind mounted to host
   directories).

#. Document your container. If your runscript doesn't supply help, write
   a ``%help`` or ``%apphelp`` section. A good container tells the user
   how to interact with it.

#. If you require any special environment variables to be defined, add
   them to the ``%environment`` and ``%appenv`` sections of the
   definition file.

#. Files should always be owned by a system account (UID lower than
   500).

#. Ensure that sensitive files like ``/etc/passwd``, ``/etc/group``, and
   ``/etc/shadow`` do not contain secrets.

#. Build production containers from a definition file instead of a
   sandbox that has been manually changed. This ensures maximal
   reproducibility, and mitigates the possibility of your production
   container being a "black box."
