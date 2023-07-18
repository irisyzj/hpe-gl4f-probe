# Deployment Overview 

The HPE GreenLake for File Storage Data Reduction Estimation Probe provides estimated data reduction rate achieable based on an example data set.  Make sure to review the [prerequisites](../prerequisites/index.md) to understand the hardware and software requirements to successfully run the probe.  This article will guide you through the process of deployment and execution of the probe.

[TOC]

## Download 

You can download the bundle from here: [935553.probe.bundle.tar.gz](935553.probe.bundle.tar.gz) and [scp](https://linux.die.net/man/1/scp) or [sftp](https://linux.die.net/man/1/sftp) to the Linux client that you wish to run the probe.

Or you may choose to use wget to download the bundle directly on the Linux client that you wish to run the probe:

```bash
wget "https://hpe-storage.github.io/gl4f-probe/deployment/935553.probe.bundle.tar.gz" -O 935553.probe.bundle.tar.gz
```

## Expand & Verify Download

Now that you've downloaded the probe, you'll need to untar it and then verify the download is correct.
```bash
export PROBE_BUILD=935553
tar -xzf ${PROBE_BUILD}.probe.bundle.tar.gz
ls -l
```
Note: image may not show current build numbers.

![probe_ls](probe_ls.png)

## Mount Filesystems Selected to Be Probed
Validated Filesystems Include, But Are Not Limited To:

* NFS
* Lustre
* GPFS
* S3 with [goofys](https://github.com/kahing/goofys)
* CIFS/SMB

For the most accurate results, do not use root-squash.

It's recommended to set read-only access on the mounted filesystem

## Create Probe Directories
Change /mnt/ to the SSD-backed local disk to be used by the probe for the hash database and logging directories
```bash
sudo mkdir -p /mnt/probe/db
sudo mkdir -p /mnt/probe/out
sudo chmod -Rf 777 /mnt/probe
```
## Size of the Data Set
* The input to the probe is a defined directory (`--input-dir`)
* The probe will automatically query the input filesystem about space consumed and file count (inodes) and use that in its calculations
* Depending on the method of mounting and underlying storage, this can often provide an inaccurate query response 
* It's highly recommended that manual estimated entries be defined for space consumed (`--data-size-gb`) and file count (`--number-of-files`) 
* These estimates do not have to be accurate, round up reasonably

## Running The Probe
The probe runs as a foreground application. This means that if your session is closed for whatever reason, the probe will stop. It's recommended running the probe as a [screen](https://linux.die.net/man/1/screen) session.

Here is an example of a command line. Edit the bold variables for the environment:

NOTE: Use underscores instead of spaces in `COMPANY_NAME` and `WORKLOAD`
```bash
export DB_DIR=/mnt/probe/db
export OUTPUT_DIR=/mnt/probe/out
export INPUT_DIR=/mnt/filesystem_to_be_probed/sub_directory
export INPUT_SIZE_GB=10000
export QTY_FILES=1000000
export COMPANY_NAME=Your_Amazing_Company
export WORKLOAD=Describe_Your_Workload
```
Start the probe: (This may take up to five minutes to start displaying output)
```bash
sudo python3 ./probe_launcher.py \
--probe-image-path ${PROBE_BUILD}.probe.image.gz \
--input-dir $INPUT_DIR \
--metadata-dir $DB_DIR \
--output-dir $OUTPUT_DIR \
--data-size-gb $INPUT_SIZE_GB \
--number-of-files $QTY_FILES \
--customer-name ${COMPANY_NAME}---${WORKLOAD}
```
 
** Example One: Small Data Sets **
To probe the directory interesting_data of 15 TB in-use and 5,000,000 files at the company ACME, the command would be:
```bash
sudo python3 ./probe_launcher.py \
--probe-image-path ${PROBE_BUILD}.probe.image.gz \
--input-dir /mnt/acme_filer/interesting_data \
--metadata-dir /mnt/data/probe/db \
--output-dir /mnt/data/probe/out \
--data-size-gb 15000 \
--number-of-files 5000000 \
--customer-name ACME---Interesting_Data
```
** Example Two: Larger Data Sets **
To probe the directory fascinating_data of 60 TB in-use and 750,000,000 files at the company FOO, and are using defined parameters for RAM and SSD-backed local disk the command would be:
```bash
sudo python3 ./probe_launcher.py \
--probe-image-path ${PROBE_BUILD}.probe.image.gz \
--input-dir /mnt/foo_filer/fascinating_data \
--metadata-dir /mnt/data/probe/db \
--output-dir /mnt/data/probe/out \
--data-size-gb 60000 \
--number-of-files 750000000 \
--customer-name FOO---Facinating_Data
```

** Example Three: Performance Throttling **
To probe the directory riviting_data of 250 TB in-use and 1,250,000,000 files at the company Initech, using defined parameters for RAM and SSD-backed local disk, but wish to have a lower performance impact on the filesystem, the command would be:

```bash
sudo python3 ./probe_launcher.py \
--probe-image-path ${PROBE_BUILD}.probe.image.gz \
--input-dir /mnt/initech_filer/riviting_data \
--metadata-dir /mnt/data/probe/db \
--output-dir /mnt/data/probe/out \
--data-size-gb 250000 \
--number-of-files 1250000000 \
--number-of-threads 4
--customer-name Initech---Riviting_Data
```
Note the `--number-of-threads` flag. By default the probe will use all CPU cores in the system but this can be used to throttle performance and reduce potential impact of the scanned filesystem.


## Other Probe Flags
While the probe is running and after completion, telemetry logs are automatically uploaded to HPE. To prevent this, add the following flag:
```bash
--dont-send-logs \
```
If you wish to send file names with the default telemetry logs, add the following flag:

```bash
--send-logs-with-file-names \
```
Probing filesystems which contain snapshots can often cause recursion issues and inaccurate results. As a result the probe automatically ignores directories named .snapshot. If your file system uses another convention, use the `--regexp-filter` command. If for some reason you want the probe to read the .snapshot directories, specify false rather than true for `--filter-snapshots`.

```bash
--filter-snapshots \    (this is the default)
```
Under most circumstances the probe should be run with adaptive chunking. However you can disable that feature by specifying this flag:
```bash
--disable-adaptive-chunking \
```
## Understanding the Results
Once started, the probe will display the current projection of potential data reduction. Once completed, the probe will display output and is further described in [Understanding Output](../output/index.md)

## Re-Running The Probe
The hash database must be empty before running the probe again:
```bash
sudo rm -r /mnt/probe/db/*
```
## Troubleshooting
Refer to the [Troubleshooting](../troubleshooting/index.md) document and contact HPE Support.














<!-- 
# TODO: delete 

Americas:
```markdown
wget "https://vastdatasupport.blob.core.windows.net/probe/935553.probe.bundle.tar.gz?sp=r&st=2023-04-20T12:46:21Z&se=2023-07-31T20:46:21Z&spr=https&sv=2021-12-02&sr=b&sig=r%2B0sacAOQ68zqS0rtXhrC9XeHK5pC08m3ImkMUE6aac%3D" -O 935553.probe.bundle.tar.gz
```

Europe:

```markdown
wget "https://vastdatasupporteuwest.blob.core.windows.net/probe/935553.probe.bundle.tar.gz?sp=r&st=2023-04-24T15:46:51Z&se=2026-04-24T23:46:51Z&spr=https&sv=2021-12-02&sr=b&sig=Nska0jbs3Fz%2BrX7IavgLBq8lZeoLyo3n2sQ%2Bz3CrdOM%3D" -O 935553.probe.bundle.tar.gz
```

Asia/Pacific:

```markdown
wget "https://vastsupportjapanwest.blob.core.windows.net/probe/935553.probe.bundle.tar.gz?sp=r&st=2023-04-24T15:47:22Z&se=2026-04-24T23:47:22Z&spr=https&sv=2021-12-02&sr=b&sig=EURsk5b4LHKM%2Bk32qyVMiab%2FZXnfIodpDiTCm5wB%2F1w%3D" -O 935553.probe.bundle.tar.gz 
```

South Africa:

```markdown
wget "https://vastsupportsanorth.blob.core.windows.net/probe/935553.probe.bun
```


The address and credentials of the target storage system must be specified in a configuration file, using the format shown in this example `storage-system.yaml` file.

```markdown
address: 10.10.10.1
username: exampleuser
password: examplepassword
```

The `address` value is either a resolvable hostname or IP address of the management interface on the storage system. The `username` value identifies a storage system user with privileges described below.

| Storage System                               | User Type   | Minimal Role   |
| :------------------------------------------- | :---------- | :------------- |
| HPE Alletra 9000, Primera, 3PAR              | System User | Browse         |
| HPE Alletra 6000, Nimble Storage             | System User | Guest          |
| HPE Alletra 6000, Nimble Storage<sup>1</sup> | Tenant      | N/A            |

<sup>1</sup> = NimbleOS 6.0 and above only.

# Command Options

| Option | Default | Description |
| :--- | :--- | :--- |
| --accept-eula | false | Confirms your acceptance of the [HPE license restrictions](../legal/eula/index.md) |
| --log.path | *None* | A file location to write log messages, in addition to stdout |
| --metrics.disable-introspection | false | Excludes metrics about the metrics provider itself, with prefixes such as `promhttp`, `process`, and `go` |
| --telemetry.addr | :8080 | The host:port address at which to provide metrics |
| --telemetry.path | /metrics | The endpoint path at which to provide metrics |

# Using an Executable File

A Linux executable file is provided through GitHub [releases](https://github.com/hpe-storage/array-exporter/releases).

When an executable file has been copied to your server, it can be invoked with this command syntax:

```markdown
hpe-array-exporter [OPTION]... CONFIG-PATH
```

Available OPTIONs are described in the [Command Options](#command_options) section.

CONFIG-PATH is the location of the storage system [configuration](#configuration) file.

## Command Example

```markdown
./hpe-array-exporter --log.path=/var/log/hpe-array-exporter.log /etc/config/storage-system.yaml
```

!!! important
    Include the `--accept-eula` option or set the environment variable `ACCEPT_HPE_STANDARD_EULA=yes` to confirm your acceptance of the [HPE license restrictions](../legal/eula/index.md).


# Using a Container Image

A container image is hosted at `quay.io/hpestorage/array-exporter:v1.0.0`, with v1.0.0 replaced by the desired release version.

When deploying the array exporter as a container, the configuration file must be mounted as a volume.

Available options, including the `--log.path` used in the example below, are described in the [Command Options](#command_options) section.

## Docker Example

In this example, the configuration file at `/tmp/storage-system.yaml` is bound to the container's `/etc/config/` directory as a volume using Docker's `-v` command option. The configuration file location inside the container is then given as a command argument. In addition, the `-p` option is used to map the container's port 8080 to port 9090 on the Docker host.

```markdown
docker run -it --name hpe-array-exporter -p 9090:8080 \
     -v /tmp/storage-system.yaml:/etc/config/storage-system.yaml \
     quay.io/hpestorage/array-exporter:v1.0.0 \
     --log.path /var/log/hpe-array-exporter.log \
     /etc/config/storage-system.yaml
```

!!! important
    Include the `--accept-eula` option or set the environment variable `ACCEPT_HPE_STANDARD_EULA=yes` to confirm your acceptance of the [HPE license restrictions](/legal/eula/index.html).

Consult the [Docker command line documentation](https://docs.docker.com/engine/reference/commandline/run/) for more information on running containers using Docker.

# Using a Kubernetes Deployment

Kubernetes deployment facilities are hosted in the [co-deployments repository](https://github.com/hpe-storage/co-deployments), including a [Helm chart](https://artifacthub.io/packages/helm/hpe-storage/hpe-array-exporter/) (via Artifact Hub) and sample [YAML files](https://github.com/hpe-storage/co-deployments/tree/master/yaml/array-exporter).
 -->