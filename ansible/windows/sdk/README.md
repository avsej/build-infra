# PREREQUISITES

This playbook likely requires at least Ansible 2.3.0. However it does currently
fail with Ansible 2.4.0 due to win_get_url not working.

The remote Windows server needs to be ready for Ansible remote
control; see README.txt in the parent directory.

Your local world needs to have Ansible installed (obviously) and configured
for controlling Windows hosts via WinRM, which means at least running

    pip install "pywinrm>=0.1.1"

See http://docs.ansible.com/ansible/intro_windows.html#installing-on-the-control-machine
for more details.

# PLAYBOOKS

playbook.yml installs everything necessary for creating a Couchbase SDK
build slave. The rest of this document refers only to this playbook.

# LOCAL FILE MODIFICATIONS NECESSARY BEFORE RUNNING PLAYBOOK

First, add any private key files necessary to the `ssh` directory. The
provided ssh/config file assumes that `buildbot_id_dsa` exists and can
be used to pull from Gerrit (necessary for commit-validation jobs), and
that a default key file such as `id_rsa` exists that can pull all private
GitHub repositories.

The `inventory` file here is a stub to show the required format. Replace at
least the IP address(es).

# RUNNING THE PLAYBOOK

The primary playbook here is `playbook.yml`.

    ansible-playbook -v -i inventory playbook.yml \
      -e ansible_password=ADMINISTRATOR_PASSWORD \
      -e vs2012_key=KEY -e vs2013_key=KEY -e vs2015_key=KEY \
      -e vs2017_key=KEY

    Note: optional ansible-playbook param to install on window server 2016:  --extra-vars "ansible_distribution=windowserver2016"
    this is to skip kb.yml install for window server 2016.
    Also note that the Visual Studio keys should have no dashes in them.

You can use our Docker image (recommended):

    docker run -it --rm -v `pwd`:/mnt \
       -v /home/couchbase/jenkinsdocker-ssh/:/ssh \
       couchbasebuild/ansible-playbook \
       -i inventory playbook.yml -e ansible_password=PASSWORD \
       -e vs2012_key=KEY -e vs2013_key=KEY -e vs2015_key=KEY \
       -e vs2017_key=KEY

# USE AS JENKINS SLAVE

The version of OpenSSH in this image trips a bug in the Jenkins
ssh-slaves plugin. There's a workaround, documented here:

https://issues.jenkins-ci.org/browse/JENKINS-42856?focusedCommentId=319018&page=com.atlassian.jira.plugin.system.issuetabpanels:comment-tabpanel#comment-319018

Namely, put the following text into the "Prefix Start Slave Command" (which
is under "Advanced" in the "Launch method" block:

    powershell -Command "cd C:\jenkins ; java -jar slave.jar" ; exit 0 ; rem '

and then put the following (a single apostrophe) into "Suffix Start Slave
Command" in the same location:

    '

# THINGS THAT COULD GO WRONG

This playbook worked on October 4, 2017. It does not specify explicit versions
of any of the toolchain requirements, because many of the packages (notably
Visual Studio 2015 itself) are specifically designed to install only the
latest version. That being the case, things could change over time to make
this playbook fail.

One likely issue is JAVA_HOME in ssh/environment, which version- and build-
specific and will need to be changed as newer JREs are released.

Another somewhat likely problem is the file `vs-unattended.xml`, which is
required to specify which Visual Studio optional packages to install. C++
support itself is an optional package, so we cannot just do a default install.
But the set of named optional packages changes over time. The Visual Studio
web installer obtains the current list via a ATOM feed. So it's possible that
this fixed vs-unattended.xml file will fall out of date. (Now that MSVC 2017
has been released, however, it seems less likely.)

If this happens, you can create a new file by:

1. Download the web installer `vs_professional.exe` from

     https://www.visualstudio.com/downloads/download-visual-studio-vs

2. Run `vs_professional.exe /CreateAdminFile C:\vs-unattended.xml` to get
   the current default. (Be sure to provide an absolute path for the .xml.)

3. Edit this file. As of this date, the key thing is to change the element
   for "Common Tools for Visual C++ 2015" to `Selected="yes"`. Currently the
   ID for that element is `NativeLanguageSupport_VCV1`.

4. (Optional) Deselect any unwanted features that are enabled by default by
   setting them to `Selected="no"`. I somewhat blindly deselected the
   following IDs:

   * BlissHidden
   * JavaScript_HiddenV11
   * JavaScript
   * PortableDTPHidden
   * Silverlight5_DRTHidden
   * StoryboardingHidden
   * WebToolsV1

