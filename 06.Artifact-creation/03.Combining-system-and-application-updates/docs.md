---
title: Combining system and application updates
taxonomy:
    category: docs
    label: tutorial
---

There are many possible strategies to combine the system and application updates you deploy to your devices. This section summarizes the crucial points you should be aware of when defining your OTA strategy and outlines the best practices we suggest for common scenarios.

## Basic concepts

### System updates using the A/B partitioning schema

When performing system updates using the A/B partitioning schema, the Mender Client writes a new file system image directly to the inactive partition. When complete, the Client verifies the checksum, updates the bootloader configuration, flips the active and inactive partitions, and reboots. On the first boot following an update, the Mender client will **commit** the update, making the bootloader configuration permanent.

One consequence of the system update is that the update will replace all the files in a filesystem, thereby deleting any new or changed files placed there. In other words, to be updatable, a file system needs to be **stateless** and contain all the software required for the device to work.

For this reason, it is common to add a dedicated, separated persistent partition to store files that must survive across full system updates. These files may include network parameters, user configuration changes, data collected and processed by the application software on the device, etc. By convention, we call this partition the *data partition*.


### Application updates

You can extend the Mender Client supporting different application-specific update patterns using the [Update Module](../08.Create-a-custom-Update-Module/docs.md) framework.

When processing an application update, the Mender Client can write files in any partition available on the device, including the system and the data partitions. However, as mentioned in the System updates section above, any changes to the system partition will be lost on system updates as the device will switch from the active to the inactive partition.

For this reason, if you designed the application update to persist across system updates, you need to ensure that all the content of application update (the files the update consist of) end up installed on the *data partition* and **not** on either of the two A/B partitions used for system updates.

Suppose the application update depends on specific libraries or capabilities of the operating system. In that case, you can express such a dependency by ensuring your system image (and updates) have a specific *Provides* entry your application update can list as part of the *Depends*. You can read more about the [*Depends* and *Provides* entries](../../02.Overview/03.Artifact/docs.md#provides-and-depends).

### Software versions reporting

The software versions the Mender Client reports to the backend are the entries ending with `.version` listed in the *Provides* fields of the Artifacts installed. When processing updates, the Client can also remove existing *Provides* entries from the local Mender database according to the *Clears Provides* entry in the Mender Artifact.

You can read more about the [Software versioning](../09.Software-versioning/docs.md) and the different options on the corresponding page.

### Drawbacks of combining system and application updates

Choosing the correct update approach should be an educated decision based on your use case, as different methods have pros and cons.

Combining system and application updates allows you to deploy multiple software versions on your device separately. However, at the same time, it suffers from increased combinatoric complexity as the software on the device is no longer represented by a single software version.

<!--AUTOVERSION: "is no longer `v%`"/ignore "system `v%`"/ignore "app1 `v%`"/ignore "app2 `v%`"/ignore-->
For example, your device software is no longer `v2.5`, but system `v2.5.0` - app1 `v1.2.4` - app2 `v2.3.1` instead.

The consequence of this complexity can contribute to the following:

* longer testing processes
* stricter API versioning and development limitations
* more complexity in customer communication (e.g., answering "what version are you running?")


## Example patterns and update scenarios

This section lists several example patterns and update scenarios where system and application updates are combined. In all the examples, the device's partitioning schema is A/B with a persistent data partition mounted on the `/data` path.

![Mender client partition layout](../../05.System-updates-Yocto-Project/01.Overview/mender_client_partition_layout.png)


### System image and application updates targeting the system partition

In this scenario, the system image provides the kernel, the operating system, the libraries required to run the applications, and the applications themselves. However, when new versions of the applications are available and do not depend on nor require a full system update, it should be possible to update them with an Update Module.

The examples assume the rootfs image contains two applications, `myapp1` and `myapp2`.

You can generate the Artifact for the full system update, including the applications, by running the command:

```bash
DEVICE_TYPE="raspberrypi4"
mender-artifact write rootfs-image -t $DEVICE_TYPE \
                                   -n rootfs-1.0 \
                                   -o rootfs-1.0.mender \
                                   -f rootfs-1.0.ext4 \
                                   --software-version 1.0 \
                                   --provides rootfs-image.myapp1.version:1.0 \
                                   --provides rootfs-image.myapp2.version:1.0
```

We can now inspect the Artifact:

```bash
mender-artifact read rootfs-1.0.mender
```
```
Reading Artifact...
........................................................................ - 100 %
Mender artifact:
  Name: rootfs-1.0
  Format: mender
  Version: 3
  Signature: no signature
  Compatible devices: '[raspberrypi4]'
  Provides group:
  Depends on one of artifact(s): []
  Depends on one of group(s): []
  State scripts:

Updates:
    0:
    Type:   rootfs-image
    Provides:
	rootfs-image.checksum: 584fc6fdf5f94c1a53c1147e620083f797d01cfed48c5e7b5131239b35363568
	rootfs-image.myapp1.version: 1.0
	rootfs-image.myapp2.version: 1.0
	rootfs-image.version: 1.0
    Depends: Nothing
    Clears Provides: ["artifact_group", "rootfs_image_checksum", "rootfs-image.*"]
    Metadata: Nothing
    Files:
      name:     rootfs-1.0.ext4
      size:     1048576
      modified: 2023-01-02 17:08:29 +0100 CET
      checksum: 584fc6fdf5f94c1a53c1147e620083f797d01cfed48c5e7b5131239b35363568
```

Please note that the rootfs Artifact provides the rootfs-image version and the versions for the two applications included in the system, `myapp1` and `myapp2`.

#### Update the applications

To generate the Artifact for the application update, you can run the command:

```bash
DEVICE_TYPE="raspberrypi4"
UPDATE_MODULE="myapps"
mender-artifact write module-image -t $DEVICE_TYPE \
                                   -n myapp1-1.1 \
                                   -o myapp1-1.1.mender \
                                   -T $UPDATE_MODULE \
                                   -f myapp1-1.1.tar.gz \
                                   --software-name myapp1 \
                                   --software-version 1.1
```

We can now inspect the Artifact:

```bash
mender-artifact read myapp1-1.1.mender
```
```
Reading Artifact...
........................................................................ - 100 %
Mender artifact:
  Name: myapp1-1.1
  Format: mender
  Version: 3
  Signature: no signature
  Compatible devices: '[raspberrypi4]'
  Provides group:
  Depends on one of artifact(s): []
  Depends on one of group(s): []
  State scripts:

Updates:
    0:
    Type:   myapps
    Provides:
	rootfs-image.myapp1.version: 1.1
    Depends: Nothing
    Clears Provides: ["rootfs-image.myapp1.*"]
    Metadata: Nothing
    Files:
      name:     myapp1-1.1.tar.gz
      size:     111
      modified: 2023-01-02 17:12:20 +0100 CET
      checksum: fa2efef86c524ede9aefb1841682bca2bb36b346b68e481c8c88c108ea2eed90
```

As you can see, the update will provide a new version for `myapp1`, which will overwrite the value previously set by the system Artifact.s

Similarly, to generate the Artifact for the other application, you can run the command:

```bash
DEVICE_TYPE="raspberrypi4"
UPDATE_MODULE="myapps"
mender-artifact write module-image -t $DEVICE_TYPE \
                                   -n myapp2-1.1 \
                                   -o myapp2-1.1.mender \
                                   -T $UPDATE_MODULE \
                                   -f myapp-1.1.tar.gz \
                                   --software-name myapp2 \
                                   --software-version 1.1
```

#### Dependencies between the system and the application updates

If the application package depends on specific capabilities provided by the system, you can define them as *Provides* and then list them in the application's Artifact as *Depends*.

For example, if support for a specific `feature1` is needed to run the application, you can specify it when writing the system Artifact:

```bash
DEVICE_TYPE="raspberrypi4"
mender-artifact write rootfs-image -t $DEVICE_TYPE \
                                   -n rootfs-1.1 \
                                   -o rootfs-1.1.mender \
                                   -f rootfs-1.1.ext4 \
                                   --software-version 1.1 \
                                   --provides rootfs-image.myapp1.version:1.1 \
                                   --provides rootfs-image.myapp2.version:1.1 \
                                   --provides rootfs-image.feature1:true
```

We can now inspect the Artifact:

```bash
mender-artifact read rootfs-1.1.mender
```
```
Reading Artifact...
........................................................................ - 100 %
Mender artifact:
  Name: rootfs-1.1
  Format: mender
  Version: 3
  Signature: no signature
  Compatible devices: '[raspberrypi4]'
  Provides group:
  Depends on one of artifact(s): []
  Depends on one of group(s): []
  State scripts:

Updates:
    0:
    Type:   rootfs-image
    Provides:
	rootfs-image.checksum: 584fc6fdf5f94c1a53c1147e620083f797d01cfed48c5e7b5131239b35363568
	rootfs-image.feature1: true
	rootfs-image.myapp1.version: 1.1
	rootfs-image.myapp2.version: 1.1
	rootfs-image.version: 1.1
    Depends: Nothing
    Clears Provides: ["artifact_group", "rootfs_image_checksum", "rootfs-image.*"]
    Metadata: Nothing
    Files:
      name:     rootfs-1.1.ext4
      size:     1048576
      modified: 2023-01-02 17:20:03 +0100 CET
      checksum: 584fc6fdf5f94c1a53c1147e620083f797d01cfed48c5e7b5131239b35363568
```

Please note the `rootfs-image.feature1` provides listed in the output above.

At this point, if the next version of the `myapp1` application requires such a feature, you can add it to the list of *Depends* when writing the Artifact:

```
DEVICE_TYPE="raspberrypi4"
UPDATE_MODULE="myapps"
mender-artifact write module-image -t $DEVICE_TYPE \
                                   -n myapp1-1.2 \
                                   -o myapp1-1.2.mender \
                                   -T $UPDATE_MODULE \
                                   -f myapp1-1.2.tar.gz \
                                   --depends rootfs-image.feature1:true \
                                   --software-name myapp1 \
                                   --software-version 1.2
```

We can now inspect the Artifact:

```bash
mender-artifact read myapp1-1.2.mender
```
```
Reading Artifact...
........................................................................ - 100 %
Mender artifact:
  Name: myapp1-1.2
  Format: mender
  Version: 3
  Signature: no signature
  Compatible devices: '[raspberrypi4]'
  Provides group:
  Depends on one of artifact(s): []
  Depends on one of group(s): []
  State scripts:

Updates:
    0:
    Type:   myapps
    Provides:
	rootfs-image.myapp1.version: 1.2
    Depends:
	rootfs-image.feature1: true
    Clears Provides: ["rootfs-image.myapp1.*"]
    Metadata: Nothing
    Files:
      name:     myapp1-1.2.tar.gz
      size:     111
      modified: 2023-01-02 17:22:12 +0100 CET
      checksum: fa2efef86c524ede9aefb1841682bca2bb36b346b68e481c8c88c108ea2eed90
```

The output of the command above lists the dependency, and it ensures the Client will reject the update if the device is not running a system that is compatible with the new version of the application.

Please note that it isn't possible to express relational comparisons using *Depends* and *Provides*, such as greater-than or lesser-than. However, if you need more granularity, you can use multiple *Provides* *keys or multiple discrete values the Artifacts can depend on instead of `true` in the example above.


### System image and application updates targeting the data partition

In this scenario, the system image provides the kernel, the operating system, and the libraries required to run the applications but not the applications themselves. The applications are installed and updated with Update Modules that write the directories and files in the *data partition*, which is persistent across system updates.

```bash
DEVICE_TYPE="raspberrypi4"
mender-artifact write rootfs-image -t $DEVICE_TYPE \
                                   -n rootfs-1.0 \
                                   -o rootfs-1.0.mender \
                                   -f rootfs-1.0.ext4 \
                                   --software-version 1.0
```

#### Deploy and update the applications

To generate the Artifact for the application update, you can run the command:

```bash
DEVICE_TYPE="raspberrypi4"
UPDATE_MODULE="myapps"
mender-artifact write module-image -t $DEVICE_TYPE \
                                   -n myapp1-1.1 \
                                   -o myapp1-1.1.mender \
                                   -T $UPDATE_MODULE \
                                   -f myapp1-1.1.tar.gz \
                                   --software-filesystem data-partition \
                                   --software-name myapp1 \
                                   --software-version 1.1
```

We can now inspect the Artifact:

```bash
mender-artifact read myapp1-1.1.mender
```
```
Reading Artifact...
........................................................................ - 100 %
Mender artifact:
  Name: myapp1-1.1
  Format: mender
  Version: 3
  Signature: no signature
  Compatible devices: '[raspberrypi4]'
  Provides group:
  Depends on one of artifact(s): []
  Depends on one of group(s): []
  State scripts:

Updates:
    0:
    Type:   myapps
    Provides:
	data-partition.myapp1.version: 1.1
    Depends: Nothing
    Clears Provides: ["data-partition.myapp1.*"]
    Metadata: Nothing
    Files:
      name:     myapp1-1.1.tar.gz
      size:     111
      modified: 2023-01-02 17:12:20 +0100 CET
      checksum: fa2efef86c524ede9aefb1841682bca2bb36b346b68e481c8c88c108ea2eed90
```

As you can see, in contrast with what we verified in the previous section, the versioning key uses the `data-partition` prefix. Because the full system update Artifacts clears provides for `rootfs-image.*`, this versioning information will be preserved by the Mender Client on full system updates, together with the file stored in the data partition.

Similarly, to generate the Artifact for the other application, you can run the command:

```bash
DEVICE_TYPE="raspberrypi4"
UPDATE_MODULE="myapps"
mender-artifact write module-image -t $DEVICE_TYPE \
                                   -n myapp2-1.1 \
                                   -o myapp2-1.1.mender \
                                   -T $UPDATE_MODULE \
                                   -f myapp2-1.1.tar.gz \
                                   --software-filesystem data-partition \
                                   --software-name myapp2 \
                                   --software-version 1.1
```

#### Dependencies between the system and the application updates

The examples from the previous section about declaring dependencies between the system and the application updates also apply to this scenario.
