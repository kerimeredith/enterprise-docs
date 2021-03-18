---
title: "Enterprise Compliance Checks"
linkTitle: "Stig Check"
weight: 2
---

#Remote Compliance Check

Anchore Enterprise remote execution manager allows the operator to run a given compliance check,
on a remote kubernetes cluster, and send the resulting output back to Enterprise for storage and correlation.
The remote execution manager will only allow the registered executables in memory to run, any other command will
result in a failure, the current registered tool is OpenSCAP. The execution manager is also responsible
for installing and uninstalling the compliance tool into the container.

## Configuration and Usage:

The operator will have to configure a few options in order to make use of REM. After which a few command will be
issued to preform a compliance check on a remote cluster.

### Pod configuration

Specify kubernete pod information in the following section:

````yaml
# Report storage location
report:
  podName: "centos"
  nameSpace: "default"
  containerName: "centos"
  reportFile: "/tmp/anchore/report.html" # pretty output report generated page.
  resultFile: "/tmp/anchore/result.xml"  # raw xml report

````



### Install compliance tool

Enable the following section in the configuration file.

```yaml
oscap:
  # This boolean flag tells REM whether or not to try to install OpenSCAP into the container (if the command is oscap)
  installEnabled: true

  # This boolean flag tells REM whether or not to try to uninstall OpenSCAP from the container
  # (after the oscap command runs and the result/report files get downloaded)
  uninstallEnabled: true


```
After the installation option has been enabled run the following command to install the compliance tool.

***note: uninstall-enabled can be set to false if you intend on leaving the tool available.***
```textmate
> rem kexec install oscap
```

### Run a compliance check

There are two options on how to run the check, the first is from the command line. The second method
is to have rem read it from the configuration file.

***From the command line***

```text
> rem kexec oscap -- xccdf eval --profile xccdf_org.ssgproject.content_profile_stig --fetch-remote-resources --result /tmp/anchore/result.xml --report /tmp/anchore/report.html /usr/share/xml/scap/ssg/content/ssg-rhel8-ds.xml
```

***From configuration file***
```yaml
command:
  # If no command is specified through arguments passed to the application on the command line, this command will be used
  # Each element of the list is interpreted as part of the command
  # I.E. echo 'hello-world' > /tmp/test.txt would look like:
  #   cmd:
  #     - echo
  #     - 'hello-world' > /tmp/tst.txt
  cmd: oscap xccdf eval --profile xccdf_org.ssgproject.content_profile_stig --fetch-remote-resources --result /tmp/anchore/result.xml --report /tmp/anchore/report.html /usr/share/xml/scap/ssg/content/ssg-rhel8-ds.xml
```

Once the check has completed the report and results should be located in the location described in the configuration under
the report section.

### Audit uploads

REM has an audit database which is used to track which compliance checks have been successfully run, this also
serves as a method to ensure fault tolerance in the case where reports have not been uploaded do to unavailable
service connections to Enterprise. REM will mark those uploads as incomplete allowing the operator to issue a flush
command and push the remainders to Enterprise.

#### Database subcommand

To list the current state for all past transactions issue the following command:
```text
> rem db list
```

To push all results which have been marked as not uploaded, issue the follow command:

***note: the --dryrun flag will show you the records which will be processed***
```text
> rem db upload
```