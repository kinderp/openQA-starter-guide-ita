# openQA-starter-guide-ita
Just a macaronic italian translation of the openQA starter guide


[[gettingstarted]]
= openQA starter guide
:toc: left
:toclevels: 6
:author: openQA Team

== Introduction

openQA is an automated test tool that makes it possible to test the whole
installation process of an operating system. It uses virtual machines to
reproduce the process, check the output (both serial console and
screen) in every step and send the necessary keystrokes and commands to
proceed to the next. openQA can check whether the system can be installed,
whether it works properly in 'live' mode, whether applications work
or whether the system responds as expected to different installation options and
commands.

Even more importantly, openQA can run several combinations of tests for every
revision of the operating system, reporting the errors detected for each
combination of hardware configuration, installation options and variant of the
operating system.

openQA is free software released under the
http://www.gnu.org/licenses/gpl-2.0.html[GPLv2 license]. The source code and
documentation are hosted in the https://github.com/os-autoinst[os-autoinst
organization on GitHub].

This document describes the general operation and usage of openQA. The main goal
is to provide a general overview of the tool, with all the information needed to
become a happy user. More advanced topics like installation, administration or
development of new tests are covered by further documents available in the
https://github.com/os-autoinst/openQA[official repository].

== Architecture
[id="architecture"]

Although the project as a whole is referred to as openQA, there are in fact
several components that are hosted in separate repositories as shown in
<<arch_img,the following figure>>.

[[arch_img]]
.openQA architecture
image::images/openqa_architecture.png[openQA architecture]

The heart of the test engine is a standalone application called
'os-autoinst' (blue). In each execution, this application creates a
virtual machine and uses it to run a set of test scripts (red).
'os-autoinst' generates a video, screenshots and a JSON file with
detailed results.

'openQA' (green) on the other hand provides a web based user
interface and infrastructure to run 'os-autoinst' in a distributed
way. The web interface also provides a JSON based REST-like API for
external scripting and for use by the worker program. Workers
fetch data and input files from openQA for os-autoinst to run the
tests. A host system can run several workers. The openQA web
application takes care of distributing test jobs among workers. Web
application and workers don't have to run on the same machine but
can be connected via network instead.

== Basic concepts
[id="concepts"]


=== Glossary

[horizontal]
.The following terms are used within the context of openQA

test modules:: an individual test case in a single perl module file, e.g.
"sshxterm". If not further specified a test module is denoted with its "short
name" equivalent to the filename including the test definition. The "full name"
is composed of the _test group_ (TBC), which itself is formed by the top-folder
of the test module file, and the short name, e.g. "x11-sshxterm" (for
x11/sshxterm.pm)

test suite:: a collection of _test modules_, e.g. "textmode". All _test
modules_ within one _test suite_ are run serially

job:: one run of individual test cases in a row denoted by a unique number for
one instance of openQA, e.g. one installation with subsequent testing of
applications within gnome

test run:: equivalent to _job_

test result:: the result of one job, e.g. "passed" with the details of each
individual _test module_

test step:: the execution of one _test module_ within a _job_

distri:: a test distribution but also sometimes referring to a _product_
(CAUTION: ambiguous, historically a "GNU/Linux distribution"), composed of
multiple _test modules_ in a folder structure that compose _test suites_, e.g.
"opensuse" (test distribution, short for "os-autoinst-distri-opensuse")

product:: the main "system under test" (SUT), e.g. "openSUSE"

job group:: equivalent to _product_, used in context of the webUI

version:: one version of a _product_, don't confuse with _builds_, e.g.
"Tumbleweed"

flavor:: a specific variant of a _product_ to distinguish differing variants,
e.g. "DVD"

arch:: an architecture variant of a _product_, e.g. "x86_64"

machine:: additional variant of machine, e.g. used for "64bit", "uefi", etc.

scenario:: A composition of
+<distri>-<version>-<flavor>-<arch>-<test_suite>@<machine>+, e.g.
"openSUSE-Tumbleweed-DVD-x86_64-gnome@64bit", nicknamed _koala_

build:: Different versions of a product as tested, can be considered a
"sub-version" of _version_, e.g. "Build1234"; *CAUTION:* ambiguity: either with
the prefix "Build" included or not

=== Jobs

One of the most important features of openQA is that it can be used to test
several combinations of actions and configurations. For every one of those
combinations, the system creates a virtual machine, performs certain steps and
returns an overall result. Every one of those executions is called a 'job'.
Every job is labeled with a numeric identifier and has several associated
'settings' that will drive its behavior.

A job goes through several states:

* *scheduled* Initial state for recently created jobs. Queued for future
  execution.
* *running* In progress.
* *cancelled* The job was explicitly cancelled by the user or was replaced by a
  clone (see below).
* *done* Execution finished.

Jobs in state 'done' have typically gone through a whole sequence of steps
(called 'testmodules') each one with its own result. But in addition to those
partial results, a finished job also provides an overall result from the
following list.

* *none* For jobs that have not reached the 'done' state.
* *passed* No critical check failed during the process. It doesn't necessarily
  mean that all testmodules were successful or that no single assertion failed.
* *failed* At least one assertion considered to be critical was not satisfied at some
  point.
* *softfailed* At least one non-critical assertion was not satisfied at some
  point (eg. a softfailure has been recorded explicitly via +record_soft_failure+)
  or workaround needles are in place.
* *incomplete* The job is no longer running but no result was provided. Either
  it was cancelled while running or it crashed.

Sometimes, the reason of a failure is not an error in the tested operating system
itself, but an outdated test or a problem in the execution of the job for some
external reason. In those situations, it makes sense to re-run a given job from
the beginning once the problem is fixed or the tests have been updated.
This is done by means of 'cloning'. Every job can be superseded by a clone which
is scheduled to run with exactly the same settings as the original job. If the
original job is still not in 'done' state, it's cancelled immediately.
From that point in time, the clone becomes the current version and the original
job is considered outdated (and can be filtered in the listing) but its
information and results (if any) are kept for future reference.

=== Needles

One of the main mechanisms for openQA to know the state of the virtual machine
is checking the presence of some elements in the machine's 'screen'.
This is performed using fuzzy image matching between the screen and the so
called 'needles'. A needle specifies both the elements to search for and a
list of tags used to decide which needles should be used at any moment.

A needle consists of a full screenshot in PNG format and a json file with
the same name (e.g. foo.png and foo.json) containing the associated data, like
which areas inside the full screenshot are relevant or the mentioned list of
tags.

[source,json]
-------------------------------------------------------------------
{
   "area" : [
      {
         "xpos" : INTEGER,
         "ypos" : INTEGER,
         "width" : INTEGER,
         "height" : INTEGER,
         "type" : ( "match" | "ocr" | "exclude" ),
         "match" : INTEGER, // 0-100. similarity percentage
      },
      ...
   ],
   "tags" : [
      STRING, ...
   ]
}
-------------------------------------------------------------------

==== Areas ====
There are three kinds of areas:

* *Regular areas* define relevant parts of the screenshot. Those must match
  with at least the specified similarity percentage. Regular areas are
  displayed as green boxes in the needle editor and as green or red frames
  in the needle view (green for matching areas, red for non-matching ones).
* *OCR areas* also define relevant parts of the screenshot. However, an OCR
  algorithm is used for matching. In the needle editor OCR areas are
  displayed as orange boxes. To turn a regular area into an OCR area within
  the needle editor, double click the concerning area twice. Note that such
  needles are only rarely used.
* *Exclude areas* can be used to ignore parts of the reference picture.
  In the needle editor exclude areas are displayed as red boxes. To turn a
  regular area into an exclude area within the needle editor, double click
  the concerning area.
  In the needle view exclude areas are displayed as gray boxes.


=== Access management

Some actions in openQA require special privileges. openQA provides
authentication through http://en.wikipedia.org/wiki/OpenID[openID]. By default,
openQA is configured to use the openSUSE openID provider, but it can very
easily be configured to use any other valid provider. Every time a new user logs
into an instance, a new user profile is created. That profile only
contains the openID identity and two flags used for access control:

* *operator* Means that the user is able to manage jobs, performing actions like
  creating new jobs, cancelling them, etc.
* *admin* Means that the user is able to manage users (granting or revoking
  operator and admin rights) as well as job templates and other related
  information (see the <<job_templates,the corresponding section>>).

Many of the operations in an openQA instance are not performed through the web
interface but using the REST-like API. The most obvious examples are the
workers and the scripts that fetch new versions of the operating system and
schedule the corresponding tests. Those clients must be authorized by an
operator using an
http://en.wikipedia.org/wiki/Application_programming_interface_key[API key] with
an associated shared secret.

For that purpose, users with the operator flag have access in the web interface
to a page that allows them to manage as many API keys as they may need. For every
key, a secret is automatically generated. The user can then configure the
workers or any other client application to use whatever pair of API key and
secret owned by him. Any client to the REST-like API using one of those API keys
will be considered to be acting on behalf of the associated user. So the API key
not only has to be correct and valid (not expired), it also has to belong to a
user with operator rights.

For more insights about authentication, authorization and the technical details
of the openQA security model, refer to the
http://lizards.opensuse.org/2014/02/28/about-openqa-and-authentication/[detailed
blog post] about the subject by the openQA development team.


=== Job groups
A job can belong to a job group. Those job groups are displayed on the index page
and in the +Job Groups+ menu on the navigation bar. From there the job group overview
pages can be accessed. Besides the test results the job group overview pages provide
a description about the job group and allow commenting.

Job groups have properties. These properties are mostly cleanup related. The
configuration can be done in the operators menu for job groups.

It is also possible to put job groups into categories. The nested groups will then
inherit properties from the category. The categories are meant to combine job groups
with common builds so test results for the same build can be shown together on
the index page.


=== Cleanup
IMPORTANT: openQA automatically deletes data that it considers "old" based on
different settings. For example job data is deleted from old jobs by the +gru+ task.

The following cleanup settings can be done on job-group-level:

[horizontal]
size limit:: Limits size of assets
keep logs for:: Specifies how long logs of a non-important job are retained after
  it finished
keep important logs for:: How long logs of an important job are retained after it
  finished
keep results for:: specifies How long results of a non-important job are retained
  after it finished
keep important results for:: How long results of an important job are retained after
  it finished

The defaults for those values are defined in
https://github.com/os-autoinst/openQA/blob/master/lib/OpenQA/Schema/JobGroupDefaults.pm[lib/OpenQA/Schema/JobGroupDefaults.pm].

*NOTE* Deletion of job results includes deletion of logs and will cause the job to
be completely removed from the database.

*NOTE* Jobs which do not belong to a job group are currently not affected by
the mentioned cleanup properties.


== Using the client script
:openqa-personal-configuration: ~/.config/openqa/client.conf

Just as the worker uses an API key+secret every user of the +client script+
must do the same. The same API key+secret as previously created can be used or
a new one created over the webUI.

The personal configuration should be stored in a file
`{openqa-personal-configuration}` in the same format as previously described for
the +client.conf+, i.e. sections for each machine, e.g. `localhost`.

== Using job templates to automate jobs creation
[id="job_templates"]

=== The problem

When testing an operating system, especially when doing continuous testing,
there is always a certain combination of jobs, each one with its own
settings, that needs to be run for every revision. Those combinations can be
different for different 'flavors' of the same revision, like running a different
set of jobs for each architecture or for the Full and the Lite versions. This
combinational problem can go one step further if openQA is being used for
different kinds of tests, like running some simple pre-integration tests
for some snapshots combined with more comprehensive post-integration tests for
release candidates.

This section describes how an instance of openQA can be configured using the
options in the admin area to automatically create all the required jobs for each
revision of your operating system that needs to be tested. If you are starting
from scratch, you should probably go through the following order:

. Define machines in 'Machines' menu
. Define medium types (products) you have in 'Medium Types' menu
. Specify various collections of tests you want to run in the 'Test suites'
  menu
. Go to the template matrix in 'Job templates' menu and decide what
  combinations do make sense and need to be tested

Machines, mediums and test suites can all set various configuration variables.
Job templates define how the test suites, mediums and machines should be
combined in various ways to produce individual 'jobs'. All the variables
from the test suite, medium and machine for the 'job' are combined and made
available to the actual test code run by the 'job', along with variables
specified as part of the job creation request. Certain variables also influence
openQA's and/or os-autoinst's own behavior in terms of how it configures the
environment for the job. Variables that influence os-autoinst's behavior
are documented in the file +doc/backend_vars.asciidoc+ in the os-autoinst
repository.

In openQA we can parametrize a test to describe for what product it will
run and for what kind of machines it will be executed. For example, a
test like KDE can be run for any product that has KDE installed, and
can be tested in x86-64 and i586 machines. If we write this as a
triples, we can create a list like this to characterize KDE tests:

  (Product,             Test Suite, Machine)
  (openSUSE-DVD-x86_64, KDE,        64bit)
  (openSUSE-DVD-x86_64, KDE,        Laptop-64bit)
  (openSUSE-DVD-x86_64, KDE,        USBBoot-64bit)
  (openSUSE-DVD-i586,   KDE,        32bit)
  (openSUSE-DVD-i586,   KDE,        Laptop-32bit)
  (openSUSE-DVD-i586,   KDE,        USBBoot-32bit)
  (openSUSE-DVD-i586,   KDE,        64bit)
  (openSUSE-DVD-i586,   KDE,        Laptop-64bit)
  (openSUSE-DVD-i586,   KDE,        USBBoot-64bit)

For every triplet, we need to configure a different instance of
os-autoinst with a different set of parameters.

=== Medium Types (products)

A medium type (product) in openQA is a simple description without any concrete
meaning. It basically consists of a name and a set of variables that
define or characterize this product in os-autoinst.

Some example variables used by openSUSE are:

* +ISO_MAXSIZE+ contains the maximum size of the product. There is a
  test that checks that the current size of the product is less or
  equal than this variable.
* +DVD+ if it is set to 1, this indicates that the medium is a DVD.
* +LIVECD+ if it is set to 1, this indicates that the medium is a live
  image (can be a CD or USB)
* +GNOME+ this variable, if it is set to 1, indicates that it is a GNOME
  only distribution.
* +PROMO+ marks the promotional product.
* +RESCUECD+ is set to 1 for rescue CD images.

=== Test Suites

This is the form where we define the different tests that we created for
openQA. A test consists of a name, a priority and a set of variables that are
used inside this particular test. The priority is used in the scheduler to
choose the next job. If multiple jobs are scheduled and their requirements for
running them are fulfilled the ones with a lower value for the priority are
triggered. The id is the second sorting key: Of two jobs with equal
requirements and same priority the one with lower id is triggered first.

Some sample variables used by openSUSE are:

* +BTRFS+ if set, the file system will be BtrFS.
* +DESKTOP+ possible values are 'kde' 'gnome' 'lxde' 'xfce' or
  'textmode'. Used to indicate the desktop selected by the user during
  the test.
* +DOCRUN+ used for documentation tests.
* +DUALBOOT+ dual boot testing, needs HDD_1 and HDDVERSION.
* +ENCRYPT+ encrypt the home directory via YaST.
* +HDDVERSION+ used together with HDD_1 to set the operating system
  previously installed on the hard disk.
* +INSTALLONLY+ only basic installation.
* +INSTLANG+ installation language. Actually used only in documentation
  tests.
* +LIVETEST+ the test is on a live medium, do not install the distribution.
* +LVM+ select LVM volume manager.
* +NICEVIDEO+ used for rendering a result video for use in show rooms,
  skipping ugly and boring tests.
* +NOAUTOLOGIN+ unmark autologin in YaST
* +NUMDISKS+ total number of disks in QEMU.
* +REBOOTAFTERINSTALL+ if set to 1, will reboot after the installation.
* +SCREENSHOTINTERVAL+ used with NICEVIDEO to improve the video quality.
* +SPLITUSR+ a YaST configuration option.
* +TOGGLEHOME+ a YaST configuration option.
* +UPGRADE+ upgrade testing, need HDD_1 and HDDVERSION.
* +VIDEOMODE+ if the value is 'text', the installation will be done in
  text mode.

Some of the variables usually set in test suites that influence openQA
and/or os-autoinst's own behavior are:

* +HDDMODEL+ variable to set the HDD hardware model
* +HDDSIZEGB+ hard disk size in GB. Used together with BtrFS variable
* +HDD_1+ path for the pre-created hard disk
* +RAIDLEVEL+ RAID configuration variable
* +QEMUVGA+ parameter to declare the video hardware configuration in QEMU

=== Machines

You need to have at least one machine set up to be able to run any
tests. Those machines represent virtual machine types that you want to
test. To make tests actually happen, you have to have an 'openQA
worker' connected that can fulfill those specifications.

* *Name.* User defined string - only needed for operator to identify the machine
configuration.

* *Backend.* What backend should be used for this machine. Recommended value is
+qemu+ as it is the most tested one, but other options (such as +kvm2usb+ or +vbox+)
are also possible.

* *Variables* Most machine variables influence os-autoinst's behavior in terms
of how the test machine is set up. A few important examples:
** +QEMUCPU+ can be 'qemu32' or 'qemu64' and specifies the architecture of the
   virtual CPU.
** +QEMUCPUS+ is an integer that specifies the number of cores you wish for.
** +LAPTOP+ if set to 1, QEMU will create a laptop profile.
** +USBBOOT+ when set to 1, the image will be loaded through an
   emulated USB stick.

=== Variable expansion

Any variable defined in Test Suite, Machine or Product table can refer to another
variable using this syntax: +%NAME%+. When the test job is created, the string
will be substituted with the value of the specified variable at that time.

For example this variable defined for Test Suite:

[source,sh]
--------------------------------------------------------------------------------
PUBLISH_HDD_1 = %DISTRI%-%VERSION%-%ARCH%-%DESKTOP%.qcow2
--------------------------------------------------------------------------------

may be expanded to this job variable:

[source,sh]
--------------------------------------------------------------------------------
PUBLISH_HDD_1 = opensuse-13.1-i586-kde.qcow2
--------------------------------------------------------------------------------

=== Variable precedence

It's possible to define the same variable in multiple places that would all be
used for a single job - for instance, you may have a variable defined in both
a test suite and a product that appear in the same job template. The precedence
order for variables is as follows (from lowest to highest):

* Product
* Machine
* Test suite
* API POST query parameters

That is, variable values set as part of the API request that triggers the jobs will
'win' over values set at any of the other locations.

If you need to override this precedence - for example, you want the value set in
one particular test suite to take precedence over a setting of the same value from
the API request - you can add a leading + to the variable name. For instance, if
you set ++VARIABLE = foo+ in a test suite, and passed +VARIABLE=bar+ in the API
request, the test suite setting would 'win' and the value would be foo.

If the same variable is set with a + prefix in multiple places, the same precedence
order described above will apply to those settings.

[[get-testing]]
== Testing openSUSE or Fedora

An easy way to start using openQA is to start testing openSUSE or Fedora as they
have everything setup and prepared to ease the initial deployment. If you want
to play deeper, you can configure the whole openQA manually from scratch, but
this document should help you to get started faster.

=== Getting tests

First you need to get actual tests. You can get openSUSE tests and needles (the
expected results) from
https://github.com/os-autoinst/os-autoinst-distri-opensuse[GitHub]. It belongs
into the +/var/lib/openqa/tests/opensuse+ directory. To make it easier, you can just
run

[source,sh]
--------------------------------------------------------------------------------
/usr/share/openqa/script/fetchneedles
--------------------------------------------------------------------------------

Which will download the tests to the correct location and will set the correct
rights as well.

Fedora's tests are also in https://pagure.io/fedora-qa/os-autoinst-distri-fedora[git]. To
use them, you may do:

[source,sh]
--------------------------------------------------------------------------------
cd /var/lib/openqa/share/tests
mkdir fedora
cd fedora
git clone https://pagure.io/fedora-qa/os-autoinst-distri-fedora.git
./templates --clean
cd ..
chown -R geekotest fedora/
--------------------------------------------------------------------------------

=== Getting openQA configuration

To get everything configured to actually run the tests, there are plenty of
options to set in the admin interface. If you plan to test openSUSE Factory, using
tests mentioned in the previous section, the easiest way to get started is the
following command:

[source,sh]
--------------------------------------------------------------------------------
/var/lib/openqa/share/tests/opensuse/products/opensuse/templates [--apikey API_KEY] [--apisecret API_SECRET]
--------------------------------------------------------------------------------

This will load some default settings that were used at some point of time in
openSUSE production openQA. Therefore those should work reasonably well with
openSUSE tests and needles. This script uses +/usr/share/openqa/script/load_templates+,
consider reading its help page (+--help+) for documentation on possible extra arguments.

For Fedora, similarly, you can call:

[source,sh]
--------------------------------------------------------------------------------
/var/lib/openqa/share/tests/fedora/templates [--apikey API_KEY] [--apisecret API_SECRET]
--------------------------------------------------------------------------------

Some Fedora tests require special hard disk images to be present in
+/var/lib/openqa/share/factory/hdd/fixed+. The +createhdds.py+ script in the
https://pagure.io/fedora-qa/createhdds[createhdds]
repository can be used to create these. See the documentation in that repo
for more information.

=== Adding a new ISO to test

To start testing a new ISO put it in +/var/lib/openqa/share/factory/iso+ and call
the following commands:

[source,sh]
--------------------------------------------------------------------------------
# Run the first test
/usr/share/openqa/script/client isos post \
         ISO=openSUSE-Factory-NET-x86_64-Build0053-Media.iso \
         DISTRI=opensuse \
         VERSION=Factory \
         FLAVOR=NET \
         ARCH=x86_64 \
         BUILD=0053
--------------------------------------------------------------------------------

If your openQA is not running on port 80 on 'localhost', you can add option
+--host=http://otherhost:9526+ to specify a different port or host.

WARNING: Use only the ISO filename in the 'client' command. You must place the
file in +/var/lib/openqa/share/factory/iso+. You cannot place the file elsewhere and
specify its path in the command.

For Fedora, a sample run might be:

[source,sh]
--------------------------------------------------------------------------------
# Run the first test
/usr/share/openqa/script/client isos post \
         ISO=Fedora-Everything-boot-x86_64-Rawhide-20160308.n.0.iso \
         DISTRI=fedora \
         VERSION=Rawhide \
         FLAVOR=Everything-boot-iso \
         ARCH=x86_64 \
         BUILD=Rawhide-20160308.n.0
--------------------------------------------------------------------------------

More details on triggering tests can also be found in the
<<UsersGuide.asciidoc#usersguide,Users Guide>>.


== Pitfalls

Take a look at <<Pitfalls.asciidoc#pitfalls,Documented Pitfalls>>.
