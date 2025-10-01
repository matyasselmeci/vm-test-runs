Running Tests as VM Jobs
========================

- [Setting up credentials](#setting-up-credentials)
- [Running osg-test in VM Universe](#running-osg-test-in-vm-universe)
- [Troubleshooting](#troubleshooting)
  - [Missing unicode fonts](#missing-unicode-fonts)
  - [Interactively connecting to a VM](#interactively-connecting-to-a-vm)

This repository drives the OSG Software nightly tests.
Whenever updating [osg-run-tests](osg-run-tests), make sure to update the copy in `/usr/bin/` on `osg-sw-submit`.

Setting up credentials
----------------------

**Note:** This one time setup must be performed before submitting any test jobs.

1.  Open a Freshdesk ticket requesting an account on `osg-sw-submit.chtc.wisc.edu` and assign it to the Software group
2.  Once the ticket has been resolved, verify access by SSHing to `osg-sw-submit`.

Running osg-test in VM Universe
-------------------------------

The procedure explained in this section replaces [this](https://opensciencegrid.github.io/technology/software/development-process/) if and only if there are functional tests for each package being tested.

1.  From `osg-sw-submit.chtc.wisc.edu`, make and populate a test run directory:

        [user@osg-sw-submit ~]$ osg-run-tests '<RUN DESCRIPTION>'

2.  Change to the test run directory (see output of osg-run-tests)
3.  If you need to change `osg-test`, make changes to a github fork of osg-test and prepend the source lines in the yaml
    parameters files (see below) with the following: `<GITHUB ACCOUNT>:<BRANCH OF OSG-TEST>;`
4.  If there are test failures that shouldn't be marked as failures in the reporting, edit `test-exceptions.yaml` add
    test failures to ignore with the following format:

        # - [test_function, test_module, start date, end date].
        [test_04_trace, test_55_condorce, 2014-12-01, 2015-01-14]

5.  To change test run parameters, edit `parameters.d/*.yaml`, or add/remove yaml files with the same format.
    Each file in `parameters.d` generates an osg-test run for every possible combination of the `platforms`, `sources`,
    and `package_sets` parameters in that file.
    1.  To change the distribution, modify the `platforms` section. Accepted values are listed below:

              platforms:
                - centos_6_x86_64
                - rhel_6_x86_64
                - sl_6_x86_64
                - centos_7_x86_64
                - rhel_7_x86_64
                - sl_7_x86_64

    2.  To change the repos that packages are installed from, edit the sources section, which has the following format:

            [<GITHUB ACCOUNT>:<BRANCH OF OSG-TEST>;] <INITIAL OSG VERSION>; <INTIAL YUM REPO> [>] [<UPDATE OSG VERSION>/][<UPDATE YUM REPO>]
            # Run osg-test from the 'opensciencegrid' github account using the 'master' branch (<https://github.com/opensciencegrid/osg-test.git>) with packages from 3.2-testing
            opensciencegrid:master; 3.2; osg-testing
            # Run osg-test (from 3.1-minefield) with packages from 3.1-release
            3.1; osg
            # Run osg-test (from 3.1-minefield) with packages from 3.1-testing that are then upgraded to 3.2-testing
            3.1; osg-testing > 3.2/osg-testing
            # Run osg-test (from 3.2-minefield) with packages from 3.2-release and 3.2-testing that are then upgraded to 3.3-testing and 3-3-upcoming-testing
            3.2; osg, osg-testing > 3.3/osg-testing, osg-upcoming-testing

    3. The `package_sets` section controls the packages that are installed, the label used for reporting,
       and whether or not SELinux is enabled:

            package_sets:
            #### Required ####
            # label - used for reporting, should be consistent across param files
            # packages - list of packages to install in the test run
            #### Optional ####
            # selinux - enable SELinux for the package set, otherwise Permissive mode (default: True)
            ##################
            - label: All
              packages:
                - osg-tested-internal
            - label: HTCondor
              packages:
                - condor.x86_64
                - osg-ce-condor

    4. The test infrastructure can read multiple yaml files in `parameters.d`, which allows you to run different,
       mutually exclusive tests.
       For example, if you wanted to test 3.4 development on EL7 in addition to the standard tests, you could add the
       following to a file in `parameters.d`:

            platform:
              - centos_7_x86_64
              - sl_7_x86_64

            sources:
              - opensciencegrid:master; 3.4; osg-development

            package_sets:
              - label: All
                packages:
                  - osg-tested-internal
              - label: All (3.4)
                packages:
                  - osg-tested-internal
              - label: HTCondor
                packages:
                  - condor.x86_64
                  - osg-ce-condor
              - label: GridFTP
                packages:
                  - osg-gridftp

6.  Submit the test run (you must submit the DAG from the run directory):

        [user@osg-sw-submit run-20200327-0925]$ ./master-run.sh

7.  Monitor the test run (as desired):

        [user@osg-sw-submit run-20200327-0925]$ condor_q -dag

8.  When the test run finishes, an email will go out to members of the software team (hardcoded in `email-analysis`).

Troubleshooting
---------------

### Missing Unicode Fonts

The HTML reports for the testing results utilize unicode characters (namely the padlock to represent that SELinux was enabled). If these characters are appearing as the character code in a block, that means that the font you're using does not support these characters. To render these characters properly, perform the following steps:

1.  Download a suitable Unicode Emoji font. We have had success with the "Noto Emoji" font available from <https://www.google.com/get/noto/>
2.  Create a `~/.fonts/` dir if one does not already exist
3.  Copy the `.ttf` file from the downloaded font `.zip` file into `~/.fonts/`
4.  Run `fc-cache -f`

### Interactively connecting to a VM

Unfortunately, VM Universe jobs don't have the ssh\_to\_job capacity that's available to other Condor jobs so if we need to investigate test failures in VMU, we'll have to spin up our own VM's. We can do this by taking the images that Neil automatically generates and create new ones from them that don't automatically run osg-test (Right now, Neil has to manually copy over the updated images. Eventually, we should have an automated way to get the most updated VMU images). Here are the steps you need to follow to set up your own VM:

1.  Grab the make-interactive-image from git:

        git clone https://github.com/opensciencegrid/vm-test-runs.git
2.  Run `make-interactive-image` using the flavor and version of Linux you need, VMU images are in `/staging/osg-images/` (NOTE: the output image needs to be in a directory that's readable by the `qemu` user):

        vm-test-runs/make-interactive-image /staging/osg-images/<INPUT IMAGE> <OUTPUT IMAGE>

3.  Make a copy of `libvirt-template.xml` and edit the `@DOMAIN@` and `@IMAGEFILE@` to a name that will be used by virsh and the path to the image file you created in the previous step
4.  Define and start the VM with your copy of the xml file:

        virsh create <XML FILE>
5.  Connect to the VM (consult BrianL, Mat or Carl for the password). You can use either the domain name or the ID returned by `virsh list`:

        virsh console <DOMAIN>
6.  Cleanup the VM. You can use either the domain name or the ID returned by `virsh list`:

        virsh destroy <DOMAIN>

### `list-rpm-versions`

This script is for listing rpm versions installed in an osg-test job output
or summarizing across an entire VMU run on osg-sw-submit.  A copy is installed
there under `/usr/local/bin`.

Below are some use cases for reference / appetite whetting.

**TL;DR:** The most common use case will probably be the one at the end with `--summarize` and `--list-outputs` (`-sl` for short) run against the timestamp for a VMU run dir.

---

Usage & Options summary:
```
[edquist@osg-sw-submit ~]
$ list-rpm-versions --help

Usage:
  list-rpm-versions [options] output-001 [packages...]
  list-rpm-versions [options] [--summarize] [run-]20161220-1618 packages...
  list-rpm-versions [options] VMU-RESULTS-URL packages...

List version-release numbers for RPMs installed in an osg-test run output
directory, as found in output-NNN/output/osg-test-*.log

The output argument can also be a root.log from a koji/mock build,
or the raw output of an 'rpm -qa' command, or an osg-profile.txt from
osg-system-profiler.

If any packages are specified, limit the results to just those packages.

Patterns can be specified for package names with the '%' character, which
matches like '*' in a shell glob pattern.

If a run directory (or, just the timstamp string) is specified, summary
information will be printed for the listed packages across all output-NNN
subdirectories for that set of osg test runs.

If a VMU-RESULTS-URL is provided, the corresponding run dir will be used.
Eg: "https://osg-sw-submit.chtc.wisc.edu/tests/20180604-1516/005/osg-test-20180604.log"
for an individual output job (005),
or: "https://osg-sw-submit.chtc.wisc.edu/tests/20180604-1516/packages.html"
for a summary of all jobs for the run.

Options:
  -A, --no-strip-arch  don't attempt to strip .arch from package names
  -D, --no-strip-dist  don't attempt to strip .dist tag from package releases

  -s, --summarize      summarize results for all output subdirs
                       (this option is implied if the argument specified is of
                       the format [run-]YYYYMMDD-HHMM)
  -l, --list-outputs   list output numbers (summarize mode only)
  -L, --max-outputs N  list at most N output numbers per NVR (-1 for unlimited)
```


Example run on a single `output-NNN` dir for all packages:
```
[edquist@osg-sw-submit /osgtest/runs/run-20161221-0423]
$ list-rpm-versions output-123 

Package                         output-123
-------                         ----------
CGSI-gSOAP                      1.3.10-1
GConf2                          3.2.6-8
apache-commons-cli              1.2-13
apache-commons-codec            1.8-7
apache-commons-collections      3.2.1-22
apache-commons-discovery        2:0.5-9
apache-commons-io               1:2.4-12
apache-commons-lang             2.6-15
apache-commons-logging          1.1.2-7
apr                             1.4.8-3
apr-util                        1.5.2-6
atk                             2.14.0-1
audit-libs-python               2.4.1-5
avalon-framework                4.3-10
...
```


Example run on a single `output-NNN` dir for two packages:
```
[edquist@osg-sw-submit /osgtest/runs/run-20161221-0423]
$ list-rpm-versions output-123 condor java-1.7.0-openjdk

Package             output-123
-------             ----------
condor              8.5.8-1.osgup
java-1.7.0-openjdk  1:1.7.0.121-2.6.8.0
```


Example run in summary mode over all `output-NNN` subdirs for a run set:
```
[edquist@osg-sw-submit ~]
$ list-rpm-versions -s 20161221-0423 condor java-1.7.0-openjdk

Package             Version-Release      Count
-------             ---------------      -----
condor              -                    5
condor              8.4.9-1              63
condor              8.4.10-1             105
condor              8.5.7-1.osgup        42
condor              8.5.8-1.osgup        79

java-1.7.0-openjdk  -                    5
java-1.7.0-openjdk  1:1.7.0.121-2.6.8.0  121
java-1.7.0-openjdk  1:1.7.0.121-2.6.8.1  168
```


Same thing, but list the output dir numbers also:
```
[edquist@osg-sw-submit ~]
$ list-rpm-versions -sl 20161221-0423 condor java-1.7.0-openjdk

Package             Version-Release      Count  Output-Nums
-------             ---------------      -----  -----------
condor              -                    5      075,078,080,082,083
condor              8.4.9-1              63     000,001,002,003,004,005,006,...
condor              8.4.10-1             105    007,008,009,010,011,012,013,...
condor              8.5.7-1.osgup        42     021,022,023,024,025,026,027,...
condor              8.5.8-1.osgup        79     028,029,030,031,032,033,034,...

java-1.7.0-openjdk  -                    5      075,078,080,082,083
java-1.7.0-openjdk  1:1.7.0.121-2.6.8.0  121    000,001,002,003,004,005,006,...
java-1.7.0-openjdk  1:1.7.0.121-2.6.8.1  168    126,127,128,129,130,131,132,...
```

