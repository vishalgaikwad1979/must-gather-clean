# Overview

`must-gather-clean` can be used for obfuscating and omitting sensitive data from [must-gathers](https://github.com/openshift/must-gather).

# Key Features

`must-gather-clean` is designed to be fast and extensible, yet simple to use. It has the following key features:
* Obfuscation for many common types of confidential information, for example IPs, MACs, DNS
* Granular control over what files should be shared
* Replace confidential information consistently to preserve debuggability
* Ability to parse and understand Kubernetes and OpenShift resources
* Concise and feature rich tool configuration
* Comprehensive reporting and reproducible obfuscation
* Community supported, contributions are welcome

# Installation

You can download the latest version of `must-gather-clean` from the [GitHub Release](https://github.com/openshift/must-gather-clean/releases) page. We currently support Linux, Mac and Windows (all ADM64+ARM64).

Unpack the binary that you downloaded, for Linux the tar file can be extracted with:
```sh 
$ tar xzf must-gather-clean-linux-amd64.tar.gz
```

Optionally, you can validate against the checksum file found on the release page, here using the linux binary:
```sh 
$ echo "$(<SHA256_SUM) must-gather-clean" | sha256sum --check
```

On Linux, you can add the binary to the path with:
```sh
$ chmod +x must-gather-clean
$ sudo mv ./must-gather-clean /usr/local/bin/must-gather-clean
```

Then you should be able to run:
```sh
$ must-gather-clean version
```

## Installing from source

Building this tool requires [Golang 1.16](https://golang.org/dl/) or later and GNU `make`.

1. Clone the repository
```sh
$ git clone https://github.com/openshift/must-gather-clean
$ cd must-gather-clean
```
2. Run make to build the tool
```sh
$ make
```
3. Check the build version to verify it was built properly
```sh
$ ./must-gather-clean version
```

# Running must-gather-clean

`must-gather-clean` should be pointed to the root folder of an already generated must-gather.

Provided a must-gather generated by the command:

```sh
$ oc adm must-gather --dest-dir=must-gather-output
```

Then cleaning can be done by running:

```sh
$ must-gather-clean -c config.yaml -i must-gather-output -o must-gather-output-cleaned
```

The cleaned must-gather can then be found in the `must-gather-output-cleaned` folder, indicated by the `-o` argument.

The configuration passed via `-c` is explained in the below [Configuration](#configuration) section.

By default, the tool runs using multiple threads and is designed to utilize the whole CPU. The number of threads can be adjusted any time with the `-w` argument, defaulting to the number of CPU cores available on the host.


## Pipe Support

The tool also supports piping content on your shell:

```sh
$ echo "some ip 192.168.2.1" | must-gather-clean 
some ip x-ipv4-0000000001-x
``` 

By default, this will obfuscate IPs and MAC addresses. You can still pass configuration options as explained in the below [Configuration](#configuration) section to further define what needs to be obfuscated. Omissions are not supported when supplying content by pipes.

# Configuration

## TL;DR

A very basic default configuration you can supply as the above `-c` flag for OpenShift can be found
under [examples/openshift_default.yaml](examples/openshift_default.yaml). If you want to obfuscate your domain names (for example DNS entries), then you have to adjust the list of `domainNames` to include yours.

In case you don't need to share networking or SDN information in the must-gather, you can run the configuration under [examples/openshift_omit_network.yaml](examples/openshift_omit_network.yaml).
This will ignore the largest log files that also take a long time to obfuscate.

## Schema

In general the schema consists of two major sections:
* [Omission](#omission)
* [Obfuscation](#obfuscation)

The omission section is used to define the omission behaviour, so to define what files to include and what not. The obfuscation section is then used to determine, on each of the included files, what content to replace (detection) and how (replacement).

The different types are explained along examples below. The whole schema itself is defined in [JSON schema](https://json-schema.org/) and
can be found in [schema.json](pkg/schema/schema.json) with more examples and documentation for each property. A more browsable
alternative can be found on [json-schema.app](https://json-schema.app/view/%23?url=https%3A%2F%2Fraw.githubusercontent.com%2Fopenshift%2Fmust-gather-clean%2Fmain%2Fpkg%2Fschema%2Fschema.json).

## Obfuscation

You can define obfuscators as a list of "types", usually customized by a couple of parameters.
The obfuscators will then be applied to each non-omitted file and on a line-by-line basis in case of text files.

The following obfuscation types are supported:

* [MAC address](#mac-address-obfuscation)
* [IP address](#ip-address-obfuscation)
* [Domain name](#domain-name-obfuscation)
* [Keywords](#keywords)
* [Regex](#regex)

### MAC address obfuscation

A minimal working example with MAC address obfuscation can be defined as following:

```
config:
  obfuscate:
  - type: MAC
```

This configuration applied on a `must-gather` folder will detect all MAC addresses recursively in all the files.
Since obfuscation is about replacing the found information, the above will simply replace the found MAC address with `xx:xx:xx:xx:xx:xx`. We call this a `Static` replacement, which is the default for all types of obfuscators.

There is another type of replacement called `Consistent`, that can be configured like this:

```
config:
  obfuscate:
  - type: MAC
    replacementType: Consistent
```

This will detect all MAC addresses and replace them with a "consistent" identifier that looks like this `xxx-mac-000001-xxx`.
For example, one of your network interfaces has the mac address `52:54:00:5e:ee:c6` and was logged, then `must-gather-clean`  will guarantee that it will always be assigned the same obfuscated consistent identifier across all files in a must-gather.
That primarily helps our support and engineers to ensure we can still understand and reproduce challenges that you were facing without putting your classified information at risk.

### IP address obfuscation

Another obfuscation type named `IP` can be used to clean IP addresses, we support both IPv4 and IPv6 except for the usual local interfaces (`127.0.0.1`, `0.0.0.0` and `::1`) that will always be preserved.

You can configure this along with the MAC obfuscator like this:

```
config:
  obfuscate:
  - type: MAC
    replacementType: Consistent
  - type: IP
    replacementType: Consistent
```

On a line-by-line basis, this will always execute the MAC obfuscation first and then the IP obfuscator - we'll go through this behaviour in more detail in the following [Chaining obfuscators and side effects](#chaining-obfuscators-and-side-effects) section.

Another configuration flag that we support for each obfuscation type is the `target`. The target is useful when the confidential information can be found not only in the file content, but also in the folder or file names.
This can very frequently happen with IP addresses, for example, through node names. You can control that independently for each type as following:

```
config:
  obfuscate:
  - type: MAC
    replacementType: Consistent
    target: FileContents
  - type: IP
    replacementType: Consistent
    target: FilePath
```

The MAC obfuscator would work on file content whereas the IP obfuscator would work only on FilePaths. There is a mixed target called `All`, that will obfuscate on both paths and contents.
The default if no target is specified is `FileContents`. It is, thus, always recommended to use the IP obfuscator with `target: All` to not accidentally leak IP information through folder names.

### Domain name obfuscation

The third built-in type of obfuscation is `Domain`, let's take a look how this can be configured:

```
config:
  obfuscate:
  - type: Domain
    domainNames:
    - "rhcloud.com"
    - "dev.rhcloud.com"
```

As you can see, this type must be customized by supplying domain names.
Kubernetes resources are defined along with their domain names (for example "apps.openshift.io/v1") and thus would be automatically recognized as such and obfuscated as a false-positive.
We thus kindly ask the user to supply their confidential domain names manually through the configuration.

The above definition will obfuscate `rhcloud.com` as `domain0000001` (consistent) or as `obfuscated.com` (static).
Note that this does not include subdomains, they would need to be separately obfuscated.
A domain name defined as `staging.rhcloud.com` would only be obfuscated as `staging.domain0000001`, thus, you should include all subdomains you want to have obfuscated (for example `dev.rhcloud.com`) in the list as well. The tool will sort them based on their specificity, so the most specific domain name will always be obfuscated first, for example `dev.rhcloud.com` will always come before `rhcloud.com` - irrespective of the order of definition.

### Custom Obfuscations

Aside from the above three built-in types to obfuscate, we also offer custom obfuscators that allow users to fine-tune the replacement of certain strings. This can be useful for custom auth token formats, confidential domain knowledge or keyword and can be customized through those two types:
* [Keywords](#keywords)
* [Regex](#regex)

#### Keywords

```
config:
  obfuscate:
  - type: Keywords
    replacement:
       hello: bye
       tomorrow: yesterday
```

This configuration will simply replace the strings on the left-hand side with the values on the right-hand side.
The `target` variable here is supported as well, so you can also target specific files and obfuscate their name, for example:

```
config:
  obfuscate:
  - type: Keywords
    target: FilePath
    replacement:
       namespaces: virtual-cluster
```

which would replace the ubiquitous "namespaces" folder to be called "virtual-cluster".

FilePath obfuscation works on the whole path (including the file name), so you can even obfuscate multi-level folder structures like this:

```
config:
  obfuscate:
  - type: Keywords
    target: FilePath
    replacement:
       namespaces/kube-system/apps: virtual-cluster/apps
```
which would condense the `namespaces/kube-system/apps` folder to become `virtual-cluster/apps` and all of its files would be under that new folder.

Since the replacement is already supplied, configuring the `replacementType` will have no effect.

#### Regex

Another common approach to detect strings by their format is using regular expressions. Internally this uses the [Golang regexp package](https://pkg.go.dev/regexp) if you need further details on how to express a pattern.

Let's take a brief example on both FilePath and FileContents:

```
config:
  obfuscate:
  - type: Regex
    target: FilePath
    regex: "release-4\..*\/ingress_controllers\/.*\/haproxy.*"
  - type: Regex
    target: FileContents
    regex: ".*ssl-min-ver TLSv1.2$"
```

The first regex would match a path like: `release-4.1/ingress_controllers/something/haproxy.log` and would `x` out the whole path.
The resulting filename would literally be: `xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`, thus there is very little practical use for using the regex like that.

This however, is much more useful in the second example where we want to obfuscate that we were using TLSv1.2 as the min version - which would also be replaced as `xxxxxxxxxxxxxxxxxxxxxxx`.

There is currently no support for consistent replacement as in the built-in types, there is a feature upcoming for capture groups and individual replacements thereof.


#### Chaining obfuscators and side effects

As seen above, obfuscators can be chained and are guaranteed to be executed in order of definition on a line of text. This can cause some interesting side effects you should be aware of.
Let's take the following contrived example to illustrate:
```
config:
  obfuscate:
  - type: Keywords
    replacement:
       a: b
  - type: Keywords
    replacement:
       b: 192.168.2.1
  - type: IP
    replacementType: Static       
```

Running the above obfuscation on the string `a wonderful evening to go dancing` would yield the following result:
`xxx.xxx.xxx.xxx wonderful evening to go dxxx.xxx.xxx.xxxncing`, which might be counter-intuitive.
So what happened here? First off, we would replace all `a` with a `b`, that `b` in turn would be replaced with `192.168.2.1` that later matches as an IPv4 and gets obfuscated in a static manner.

You can ensure that this does not happen, by providing custom obfuscators at the very bottom of the definition, preferably after all built-ins, and by ensuring you match on very specific terms (for example by supplying word boundaries in regular expressions).

## Omission

To ensure certain files will never be shared, must-gather-clean helps you to omit files.
There is support for the most common types of files in the must-gather:
* [File Pattern](#file-pattern)
* [Kubernetes Resource](#kubernetes-resource)

### File Pattern

File patterns can be useful to omit certain folders and files by their path. The paths are always relative to the root of the must-gather that was supplied by the "-i" flag.

```
config:
  omit:
  - type: File
    pattern: "*/namespaces/openshift-sdn/pods/*/*/*/logs/*.log"
```

This example illustrates how the globbing of the path works, you need to supply the respective folder structure down to the specific files you want to omit. A simple `*.log` will not suffice to omit all files matching the log extension in the must-gather.
The rationale here is to be explicit about what is being omitted to later avoid chasing the accidentally missing files.

For more information about the supported syntax refer to the documentation on [filepath.Match](https://pkg.go.dev/path/filepath#Match) that powers this feature.

### Kubernetes Resource

Most of the very confidential information, for example authentication tokens and certificates, are stored in Kubernetes resources and its yaml representation. We added the ability to omit those resources by their familiar Kubernetes resource identifiers.

You can configure to omit all secrets by its kind as follows:

```
config:
  omit:
  - type: Kubernetes
    kubernetesResource:
       kind: "Secret"
```

which will, irrespectively of its API version and namespace, omit all Secrets that are defined in the must-gather. `kind` must always be supplied as a mandatory parameter.

To supply one or many namespaces, you can supply a list as follows:

```
config:
  omit:
  - type: Kubernetes
    kubernetesResource:
       kind: "Secret"
       namespaces: ["kube-system", "default"]
```

which will omit only the secrets in the defined namespaces.

Since those resources are usually versioned and fenced off by their own naming schema, you can supply an API version that must be omitted:

```
config:
  omit:
  - type: Kubernetes
    kubernetesResource:
       kind: CertificateSigningRequest
       apiVersion: certificates.k8s.io/v1
       namespaces: ["kube-system"]
```

### Chaining omitters

Similar to obfuscators, you can also chain the omitters. The guarantee is that each omission type will be called for each file path in order of their definition. The first omitter to match a file path is used as the final decision, subsequently defined omitters will be skipped.
To have optimal performance, it is important that the most selective omitters should be defined first, the most specific at the bottom.

## Reporting

At the end of every cleaning a `report.yaml` will be written to the current working directory. A different folder for the report can be configured by supplying the `-r` argument.

The report contains a section about the replacements:
```
replacements:
  - - canonical: 10.0.187.218
      replacedWith: x-ipv4-0000000001-x
      occurrences:
        - original: 10-0-187-218
          count: 12429
        - original: 10.0.187.218
          count: 7855
     ...
  - - canonical: 0E:A0:E7:92:3A:A3
      replacedWith: x-mac-0000000001-x
      occurrences:
        - original: 0e:a0:e7:92:3a:a3
          count: 1
     ...
```

Each replacement comes with a canonicalized version of a detected text. In the above example report you see that the IP address `10.0.187.218` was replaced with `x-ipv4-0000000001-x` much more often formatted as `10-0-187-218` - 12429 over 7855 times. Omissions are also included in the report, those will report a listing of all files that have been omitted from the output.

Please ensure to not share the report as this allows to relate the original confidential data with their obfuscated replacements.

### Reproducing runs

To reproduce runs of an already done cleaning process, you can reuse the report as a configuration. At the bottom of each report, you'll also find the initial configuration used to clean along with the reported replacements:

```
config:
    obfuscate:
      - replacement:
            10-0-187-218: x-ipv4-0000000001-x
            10.0.187.218: x-ipv4-0000000001-x
            ...
        replacementType: Consistent
        target: All
        type: IP
      - replacement:
            0a:60:54:e4:52:a5: x-mac-0000000003-x
            0a:fd:78:19:88:d7: x-mac-0000000002-x
            0e:a0:e7:92:3a:a3: x-mac-0000000001-x
        replacementType: Consistent
        target: All
        type: MAC
      - domainNames:
          - rhcloud.com
          - dev.rhcloud.com
        replacement:
            aws.dev.rhcloud.com: domain0000000001
        replacementType: Consistent
        target: All
        type: Domain
```

This allows to reproduce a run completely by passing the `report.yaml` back as a configuration:
```sh
$ must-gather-clean -c report.yaml -i must-gather-output -o must-gather-output-cleaned
```

The resulting cleaned must-gather is replaced exactly as in the previous run that created the report.

# Contributing to must-gather-clean

This project is a community supported open source project under the OpenShift umbrella. We're a small cross-functional team that initially built this tool and want to foster a community around it.

Contributions to the code and documentation are always welcome and will be reviewed in due time by a member of our team. For larger changes, we would appreciate starting a discussion in an [Github Issue](https://github.com/openshift/must-gather-clean/issues) before.

You can find more information in [CONTRIB](CONTRIB.md).

## Reporting bugs

Before filing a bug report, ensure the bug hasn't already been reported by searching through the project [Issues](https://github.com/openshift/must-gather-clean/issues). 

Please choose the `Bug Report` template when creating an issue and fill the required sections in the template - this helps us to triage the issue get it fixed faster.
