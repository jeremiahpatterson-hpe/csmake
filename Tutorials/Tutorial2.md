Tutorial 2: Creating OVAs from VMDKs

In this tutorial, you will learn the basics of how csmake does:

    Basic file tracking
    Management of modules to complete specific tasks
    Reuse of modules built for a different, but related task

This tutorial assumes you know how to operate the monolithic CloudSystem appliance builds, or can otherwise generate vmdk's that you can/want to package into ovas.

Having git installed on your system is required if you wish to get the csmake modules used in this tutorial from the cloudsystem-appliance-build repository.

Having Internet access set up on your system is required if you wish to follow steps in the appendix to build your own qemu-img and vbox-img packages.

A 64-bit Linux (amd64) debian-based distro operating system is required

This tutorial was verified against an Ubuntu 14.04.02 LTS system (YMMV on other versions/distros)
Step 1: Ensure csmake, qemu-img and vbox-img are installed

This exercise uses csmake and uses a csmake module that uses both qemu-img and vbox-img.  We use vbox-img, which is not readily available from a common apt repository, and a later version of qemu-img than what is readily available on ubuntu.  vobx-img reliably produces streamOptimized vmdk's, which works well for creating and importing ova's.  qemu-img v2.2 produces more correct qcow2 images and has the ability to produce newer qcows, but remain backwards compatible to 0.10.  qemu-img can also produce streamOptimized vmdk's, but does not do so in a manner that vmware products will accept from within an ova.

If you do not have version 1.3.2 or later of csmake, please install that now

You also need to install qemu-img and vbox-img:
http://tabasco0.usa.hp.com/release/build_tools/qemu-img-2.2.0-1.0.deb
http://tabasco0.usa.hp.com/release/build_tools/vbox-img-4.3.18-1.0.deb

(If desired, you can build these completely from source.  See appendix below)
Steps 2-4 (shortcut): Get the attached archive and expand it in "/path/to/my/ova-packager"

(/path/to/my/ova-packager is any location you choose on your system to place the contents of the archive for the purpose of doing conversions)


Download the tarball from here

Expand the tarball in what we'll call in the rest of the tutorial /path/to/my/ova-packager

(e.g.: mv ova-packager.tar.gz /path/to/my; cd /path/to/my; tar -xzvf ova-packager.tar.gz)

Skip down to Step 5
Step 2 (alternative): Create a conversion project (new directory location)

Create a new directory to do the vmdk to ova conversions (path/to/my is assumed to be any path you choose on your build system to work with)
mkdir /path/to/my/ova-packager
cd /path/to/my/ova-packager
mkdir CsmakeModules

(Note: The project source is attached as a zip file on this wiki as well so you can use this out of the box.)
Step 3 (alternative): Get the csmake modules used to do conversions

The required modules are available directly from the cloudsystem-appliance-build project (where the csmake based CloudSystem builds are defined for all images)

You can get the cloudsystem-appliance-build project and desired modules like this (with git installed on your system, /path/to/modules is where you've cloned or will clone cloudsystem-appliance-build):
cd /path/to/modules
git clone git://git.gozer.hpcloud.net/hp/cloudsystem-appliance-build
cd cloudsystem-appliance-build/CsmakeModules
cp ModifyVmdkDDB.py CreateOVA.py ConvertVirtualImage.py /path/to/my/ova-packager/CsmakeModules
cd /path/to/my/ova-packager

You can confirm you did this correctly by going back to the ova-packager directory and running "csmake --list-types --modules-path=+local"  The only modules you'll see documentation for are the modules you copied (ModifyVmdkDDB, CreateOVA, and ConvertVirtualImage) and the built-in, ~~phases~~ module.
Step 4 (alternative): Create the csmakefile

The csmakefile you create in this step will codify the process for using the modules you just brought over from cloudsystem-appliance-build.

Because creating ova's is actually not that different from building other kinds of packaging, and because you will be using the file tracking features of csmake, it is required that you define some metadata that will be referenced by the ova packaging and used by csmake.  We will start by editing a fresh csmakefile.  Add the following to the csmakefile
csmakefile Contents
# The ~~phases~~ section defines that we will be using the "build" phase
# It also documents what the build phase will do and defines that
# by default, csmake will perform a build phase only
[~~phases~~]
build=Create the ova from a vmdk
clean=Clean up ovas generated by a build
**default=build
 
# This section defines the basic metadata that will be used
# to generate the ova images.
# This section also introduces the files that will be converted:
#    By default, **files definitions will look in the source directory
#                        by default, the source directory is the "pwd"
[metadata@convert-ova-metadata]
name=convert-ova-conversion
version=8.5.0
description=A convert-ova converted vmdk
about=This image was converted using the convert-ova csmake build process
packager=CloudSystem Development Team
manufacturer=Hewlett-Packard
**files=<vmdk (vmdk-image:generated-vm-disk)> *.vmdk
 
# This section will take anything that has a purpose of a "generated-vm-disk"
# and translate it to a VMDK that can be used in an ova.
# The 'ConvertVirtualImage' module's help can be accessed by:
#    csmake --list-type=ConvertVirtualImage
#    If you look at the help, you will see that the conversion implementation
#    keys off of the *type* of the image specified in the **files (or other
#    file tracking assigned type.
# The particular **maps option used below is a little dangerous because it makes
# assumptions about where the image resides and hazards doing an overwrite
# of that file if the source image in the map resides in the same directory
# as the target.  The convert won't work as an overwrite.
# It also assumes that we're only converting vmdk's.
# It would be safer to not assume these things and ensure we write the converted
# image to a different location with a vmdk extension,
# such as %(RESULTS)s/converted/{~~filename~~}.vmdk
[ConvertVirtualImage@convert-vmdk]
**maps=<(:generated-vm-disk)> -(1-1)-> <images (vmdk-image:appliance-disk)> {~~file~~}
 
# Here we see that we intentionally do an overwrite.
# If you look at the documentation for this module, the module expects
# to have an overwrite specified in the map.
[ModifyVmdkDDB@modify-ddb]
**maps=<(:appliance-disk)> -(1-1)-> {~~path~~}/{~~file~~}
substitute_1=ddb.adapterType="ide"
    ddb.adapterType="lsilogic"
#Why 2147483647?  Because it works....
#TODO: Find a more appropriate value if one exists
append=ddb.toolsVersion="2147483647"
 
 
# Here the mapping drops the ova in the "results" directory.
# %(RESULTS)s represents the default directory for
# the right hand side of the mapping.
[CreateOVA@create-ova]
**maps=<(:appliance-disk)> -(1-1)-> <(ova)> {~~filename~~}.ova
vm-name=converted-vmdk-vm
vm-description=A converted VMDK VM
vm-cpus=4
vm-memory=4096
os=ubuntu64
os-description=hLinux cattleprod
disk-capacity-appliance-disk=200G
disk-format-appliance-disk=VMDK Stream Optimized
 
[command@]
00=convert-ova-metadata
01=convert-vmdk
02=modify-ddb, create-ova

There appears to be quite a bit in this file, but let's break it down.  Before we do that, remember that you can always do: csmake --list-type=<section type>   (for example, csmake --list-type=command) to get documentation associated with that section.  The documentation for any section should explain what options the section wants and the syntax it expects for the options. 
command and ~~phases~~ Sections

First, the "command" section.  We see it specifies several ids and an ordering for these ids.  "command" sections exist to invoke several steps as a single step (a simple functional abstraction), "command" sections can be accessed by id from the command line.  In this csmakefile, we've specified an "id-less" command implying it is the default (i.e., what is executed when csmake is invoked without a "--command") .  The "default" id will accomplish the same semantics.

A "command" section is where the execution of a csmakefile will always start.

Another thing to note up front is the "~~phases~~" section.  As the comments say, it's there simply to provide guidance on how a build should proceed via documentation and provide defaults so that if csmake is invoked without phases, it will do what the user should expect.  In this case, if csmake is invoked without any phases specified, it will just do a build phase and stop.  In fact, we can conclude from this ~~phases~~ section that a build phase is the only valid phase to execute since it's the only phase documented in the section.  The information provided in the "~~phases~~" section may be accessed by doing: csmake --list-phases

Now, back to the "command" section.  The ids in the command options are executed in order of the keys.  Here we have "00", "01", and "02" - these keys can be anything, for example "apple", "boat", and "car", just as long as it is understood that these keys will be consumed in lexicographic order.
metadata Section

The first id the "command" references is "convert-ova-metadata".  This is an id to our "metadata" section, and will do two things here.  The first is that it will set up the initial metadata for the build.  One of the goals of csmake is to allow a csmakefile to specify how to get from raw bits to a final, customer-consumable product.  "metadata" sections provide the glue for this concept that both defines what the build is trying to create, and the information that a resulting package from this build should have.  This "metadata" section also contains a **files option.  Any step in the build process can define a **files option, but it's good practice to call out as many of the source files as possible within the metadata section, in part because metadata is usually, if not always the first section to be executed for a build, but second, the sources for the build are really part of the makeup of the product definition - providing a good visual of both what you intend to build, and what files contribute to creating that build product.  If the **files definitions have relative paths, csmake will look in the working directory, which by default is the current working directory (pwd), but may be different if --working-dir is passed to csmake when it is invoked. 

Here we see that **files in the "metadata" section introduce *.vmdk into the file- tracking as "<vmdk (vmdk-image:generated-vm-disk)>" which means that the files will be grouped under the "vmdk" id, are vmdk-image type files, and the purpose (or intent as it is referred to in the man pages for csmakefile) is to use these vmdk's as generated-vm-disk.  As you can see on future sections, these files may be referred to in a number of ways, just id, just file type, just intent, or just  filename or any combination of these. 
ConvertVirtualImage Section

The next section in the "command" after "convert-ova-metadata" is: "convert-vmdk" which is a ConvertVirtualImage type of section.  The only option this section has is the "**maps" option - which tells csmake that we want to start with the files that are intended to be generated-vm-disks and we're going to map them to "<images (vmdk-image:appliance-disk)>", which means "grouped" as images, with the vmdk-image type and with the intent of use as appliance-disks.  The "{~~file~~}" just means we're mapping it to the same file and extension, but mappings, if the path is not otherwise defined, will map the file to the results or "target" directory.  The default results directory for csmake is "target" under the current working directory, this can be altered by passing a --results-dir flag to csmake. 

NOTE: If you wanted to add other kinds of images  to be processed (e.g., qcow2s or raw images) into vmdk's and then on to be placed in the ova, you could add these to the **files option in the metadata and you would want to change the mapping in the "convert-vmdk" section to be "{~~filename~~}.vmdk" from just "{~~file~~}" so that they would have the correct extension, avoiding confusion.  For more details on how mapping works, the "~~file~~", etc. substitutions, and how ConvertVirtualImage understands how to convert images from the mapping information see:  man csmakefile (FILE MAPPING AND TRACKING section) and csmake --list-type=ConvertVirtualImage (from the ova-packager directory since we're using this module as a module local to the build).  ConvertVirtualImage uses both the "from" and "to" file types to determine what kind of conversion to do and which tool to use to do the conversion (as per the documentation)
ModifyVmdkDDB Section

Following the "convert-vmdk" id is the "modify-ddb" id, which is a ModifyVmdkDDB type of section.  Because vmdk's are large, the most efficient way to handle modifying the DDB section of a vmdk file is to do it in-place, so this section wants a mapping that will do such a modification in-place and will complain if it gets a mapping that isn't in-place.  Since the mapping is in-place and the file type and intent do not change, the "to" side of the mapping is simply "{~~path~~}/{~~file~~}" which tells csmake to use the exact same path, filename and extension as it gets in the "from" side of the mapping.  If, for some reason, it would be helpful to use a new grouping for these modified images, or provide a different intent, you could do this along with the "{~~path~~}/{~~file~~}" right hand side, e.g., "<modified-images (:fixedup-appliance-disk)> {~~path~~}/{~~file~~}".

On the left side of this mapping, we just specify "<(:appliance-disk)>" which tells csmake to map anything that is intended to be an appliance-disk.  This is really a bit too loosely defined in general, because ModifyVmdkDDB's only work with vmdk's, so "<(vmdk-image:appliance-disk)>" might be a better choice simply because it will help specify correct results as this project is maintained (if it were to be maintained).

As you can see from the options to this section, we're fixing up and appending different parts of the ddb.  The changes here will align the vmdk with what CloudSystem needs in these images in order to properly work in an ESX environment.  You can get more details on how this section works by doing csmake --list-type=ModifyVmdkDDB  in the ova-packager directory where you do your builds because we are using ModifyVmdkDDB as a module local to this build environment.
CreateOVA Section

Finally, we get to the last step, creating an ova.  The very last id the "command" section calls for is "create-ova" which is defined as a CreateOVA section.  Here we see that we're taking anything that is intended to be an "appliance-disk" and mapping it into an ova with the same filename as the given appliance-disk, but with an .ova extension ({~~filename~~}.ova).  The same caution applies here as in the ModifyVmdkDDB section for the mapping, it would be more robust to map "vmdk-image:appliance-disk" just in case there are other types of appliance-disk's being tracked in the build.  The right side of the mapping (<(ova)> {~~filename~~}.ova) says we'll be creating files with an "ova" type, but same intent (which may not be entirely accurate since ova's are machine descriptions, not just disks), and same grouping id as the files mapped from the left side.

The CreateOVA implementation will pull information from the metadata and its own options to fill out the information contained in the ova about the vm the ova describes.  You can read more about what "CreateOVA" does by doing, csmake --list-type=CreateOVA  from the ova-packager directory since we're using CreateOVA as a module local to the build.
Step 5: Copy in a vmdk and run a build

Now you are ready to try to build an ova.

First, copy one or more vmdk's into your ova-packager directory

Then, run csmake
cd /path/to/my/ova-packager
cp /path/to/my/vmdks/*.vmdk .
csmake

You should see the build go by and from the build directory, under target, you will find your ova's
Tutorial 2 Appendix
Building qemu-img and vbox-img From Sources

Both the qemu-img and vbox-img projects are subdirectories under cloudsystem-appliance-build.  So, first clone cloudsystem-appliance-build:
git clone git://git.gozer.hpcloud.net/hp/cloudsystem-appliance-build
cd cloudsystem-appliance-build

Before proceeding, make sure you have some basic tools installed, like a C++ compiler, etc.
Building qemu-img

Change directories down into the qemu-img project and run csmake:
cd packaged-deliverables/qemu-img
csmake
Building vbox-img

Change directories from cloudsystem-appliance-build down into vbox-img and run csmake:
cd packaged-deliverables/vbox-img
csmake
Understanding what's going on here

In both cases, you'll see csmake perform several build phases.  These default phases and the phases that are valid to use with these projects are defined in the csmakefile ~~phases~~ section for each project.

The results for these builds will ultimately end up under the "target/debpackage" directory which will contain the resulting deb package you can use to install the tool.

These are both good examples of getting from source, driving several build steps (even in another tool), and packaging the results for consumption because they are complex enough that they are not trivial, but not so complex that they are intractable to be understood by someone new to csmake's ins and outs.  Look at the csmakefiles, try the builds, try experimenting with the various phases involved.

<sub>This material is under the GPL v3 license:

<sub>(c) Copyright 2017 Hewlett Packard Enterprise Development LP

<sub>This program is free software: you can redistribute it and/or modify it
under the terms of the GNU General Public License as published by the
Free Software Foundation, either version 3 of the License, or (at your
option) any later version.

<sub>This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the GNU General
Public License for more details.

<sub>You should have received a copy of the GNU General Public License
along with this program.  If not, see <http://www.gnu.org/licenses/>.

