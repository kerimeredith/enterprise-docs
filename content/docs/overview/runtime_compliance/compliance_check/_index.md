---
title: "Enterprise Compliance Checks"
linkTitle: "Stig Check"
weight: 2
---

#Remote Compliance Check

Anchore Enterprise remote execution manager (rem) enables an operator to run a compliance check for a defined container 
within a Kubernetes Cluster. REM contains functionality to perform package management such as installation and removal
of OpenSCAP, retrieval of generated results files that will be sent to the compliance API, and the ability to provide a
local data-store if upload functionality is disabled or unavailable.




* [Installation](#rem-installation)
* [Usage](#usage)
    * [Completion](#completion)
* [Configuration](#configuration)
* [Pod Configuration](#pod-configuration)
* [Compliance Tool Installation](#compliance-tool-installation)
* [Running a Compliance Check](#run-a-compliance-check)
* [Custom STIG Targets](#custom-stig-targets)
* [Auditing Uploads](#audit-uploads)
* [Database Utilities](#database-subcommand)
* [Contributing](#developing-rem)

### REM installation
REM can be installed various ways depending on your distribution.

#### Windows
Currently, for windows users, the best way to get REM is to download the compiled windows binary from our [releases page](https://github.com/anchore/rem/releases)

#### Homebrew
To install REM via homebrew, a github token with access to this repository and [homebrew-rem](https://github.com/anchore/homebrew-rem) is required
```
export HOMEBREW_GITHUB_API_TOKEN='xxx'
brew tap anchore/rem
brew install anchore/rem/rem
```

#### Download Release Script
A provided [bash utility script](./extra/download-release) can be used for Linux and MacOS users.

To download the latest release, just run the script:
```
./extra/download-release 
```

To download a specific version/artifact, add arguments:
```
./extra/download-release 0.1.1 rem_0.1.1_darwin_amd64.tar.gz
```

### Usage:

REM can work well out of the box with minimal required configurations.

At the very least, REM needs to be able to authenticate with the Kubernetes API, know which command to run, and know which
pod and container to connect to. If you have a Kube Config at `~/.kube/config`, REM can use that out-of-the-box.

To see how to configure REM with these minimal details, see [here](#pod-configuration)

#### Completion
REM supports shell completion for bash, zsh, and fish. Run `rem completion -h` for more information

### Configuration

REM will search for the configuration file in a few locations:

***The following examples are listed in the order of precedence.***


From the CLI you can pass a "-f" or "--config" flag with the path to the configuration file.

```text
> rem -f /tmp/anchore/config.yaml
```

Setting an Environment variable:

```text
> export REM_CONFIGPATH="/tmp/anchore/config.yaml"
```

Current directory of execution:
```text
./rem.yaml
.rem/config.yaml
```
User home directory path:
```text
~/.rem.yaml
```

XDG configured directory path:
```text
rem/config.yaml
```

It is always recommended to use the configuration file that is attached to each release as an artifact.
The [example](./rem.yaml) in the repository is a good reference for explaining which configuration key does what.

### Pod configuration

This section will describe the minimum required configuration required for REM to work.

In the file, you can specify kubernetes pod information in the following section:

````yaml
# This section tells REM the execution details for the STIG check report:
# Pod Name, Namespace, and Container Name are required so that REM knows where to exec the stig check
report:
  podName: "centos"
  nameSpace: "default"
  containerName: "centos"

  # These must be set via the file, and correspond to the command being executed in the container
  # For example, if your compliance check command looks like this:
  #   oscap xccdf eval --profile <profile> --results /tmp/anchore/result.xml --report /tmp/anchore/report.html target.xml
  # The values should for --results and --report should match the values of these configurations.
  # The file paths defined here are also where REM downloads the files from the container. You can think of it like this:
  #    docker cp container:/tmp/anchore/report.html /tmp/anhore/report.html
  reportFile: "/tmp/anchore/report.html" 
  resultFile: "/tmp/anchore/result.xml"

# REM supports Kubernetes Configuration in the following manner:
#   1. If you have a Kubeconfig at ~/.kube/config, you don't need to set any of these fields below, REM will just use that
#   2. If you want to explicitly specify kubernetes configuration details, you can do so in each field below (ignore path)
#   3. If you are running REM within Kubernetes, set path to "use-in-cluster" and set cluster to the cluster name and you don't need to set any of the other fields
kubeconfig:
  path: "" # set to "use-in-cluster" if running REM within a kubernetes container
  cluster: ""
  clusterCert: # base64 encoded cluster cert
  server:  # ex. https://kubernetes.docker.internal:6443
  user:
    type:  # valid: [private_key, token]
    clientCert: # if type==private_key, base64 encoded client cert
    privateKey: # if type==private_key, base64 encoded private key
    token: # plaintext service account token

````
As an alternative, or a way to override the setting in the configuration file on the command line, you can pass a few flags to set new values.
```
// Here, <cmd> is the full oscap command to execute within the container, and the args before the double hyphen '--' are telling REM where to run the command
$ rem kexec -n <namespace> -p <pod> -c <container> -k <kubeconfig-path-override> -- <cmd>

// Example (this will use kubeconfig at ~/.kube/config)
$ rem kexec -n default -p anchore-pod -c anchore-container -- oscap xccdf eval --profile standard --result /tmp/result.xml --report /tmp/report.html target.xml 
```
Note: The double hyphen `--` is important because it tells REM that all subsequent flags should be passed to the container command

A full list of the options supported by the `rem kexec` command can be found by running the command with the `-h` or `--help option`
i.e.
```
rem kexec --help
```


### Compliance Tool Installation

Enable the following section in the configuration file.

```yaml
command:
  .
  ..
    oscap:
      # This boolean flag tells REM whether or not to try to install OpenSCAP into the container (if the command is oscap)
      installEnabled: true
    
      # This boolean flag tells REM whether or not to try to uninstall OpenSCAP from the container
      # (after the oscap command runs and the result/report files get downloaded)
      uninstallEnabled: true


```
After the installation option has been enabled this will allow the operator to manually install the compliance tool
or allow REM to automatically install the missing tool needed to run the compliance check.

***note: uninstallEnabled can be set to false if you intend on leaving the tool available.***

Running the following will install OpenSCAP but this is not mandatory.
```textmate
> rem kexec install oscap
```

### Run a compliance check

There are two options on how to run the check, the first is from the command line. The second method
is to have rem read it from the configuration file.

***From the command line***

```text
> rem kexec oscap -- xccdf eval --profile xccdf_org.ssgproject.content_profile_stig --fetch-remote-resources --results /tmp/anchore/result.xml --report /tmp/anchore/report.html /usr/share/xml/scap/ssg/content/ssg-rhel8-ds.xml
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
  cmd: oscap xccdf eval --profile xccdf_org.ssgproject.content_profile_stig --fetch-remote-resources --results /tmp/anchore/result.xml --report /tmp/anchore/report.html /usr/share/xml/scap/ssg/content/ssg-rhel8-ds.xml
```

Once the check has completed the report and results should be located in the described path in the configured under the report section.

### Custom STIG targets

REM has the option to allow the operator to specify a custom target by setting a path under customTargetPath.

```yaml
# If a custom OSCAP profile is desired, specify it's path here
# Note: this will be placed into a /tmp/anchore/ directory in the container at runtime, so the command being executed
customTargetPath: <local path to target>/custom.xml
```


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
In order to retreive detailed information about a transaction use the db get command with the id:

```text
> rem db get 1
```

To push all results which have been marked as not uploaded, issue the follow command:

***note: the --dryrun flag will show you the records which will be processed***
```text
> rem db upload
```