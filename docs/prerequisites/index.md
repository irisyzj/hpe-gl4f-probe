# Prerequisites Overview

Before we can start deploying the HPE GreenLake for File Storage Data Reduction Estimation Probe, take a moment to review the prerequisites to understand the hardware and software requirements to successfully run the probe. This is intended for customers that are running the probe on their own infrastructure. 

[TOC]

## Hardware Minimum Requirements

Actual hardware requirements depend on the amount of data to be scanned. Examples on how to scope hardware based on dataset size are provided at the end of this page.

* 16 CPU cores or higher Intel Broadwell-compatible or later CPUs
    * The Probe requires CPU instructions that are not available on older CPUs
    * The Probe will run virtually on Intel based hardware that has a Virtual Cluster vMotion minimum compatibility of Intel Broadwell-compatible or later
    * The Probe has not been evaluated on AMD CPUs
* 128 GB RAM or higher 
    * The probe consumes almost 100GB of RAM upon launch
    * The more RAM, the better the Probe will perform and the more data can be scanned
* 10 GbE Networking or higher
* 50 GB SSD-backed local storage or higher (NVMe or FC/iSCISI LUNs)
   * This local SSD capacity is needed for the database the probe builds and logging
   * Must be equivalent to 0.6% of the data to be scanned
   * Disk storage must have very high sustained IOPs
   * The larger the local SSD allocated, the more data can be scanned
   * Local SSD filesystem should be ext4 or xfs

## Operating System Minimum Requirements

We've tested the following, but most modern Linux distributions should be fine:

* Ubuntu 18.04, 20.04
* Centos/RHEL 7.4+
* Rocky/RHEL 8.3+

## Software Requirements

* Docker: 17.05 +
* python3 (for launching the probe)
* screen (for running the probe in the background)
* wget (for downloading the probe image)

## Sample Data Set Filesystem Requirements 

Be aware that if the filesystem has atime enabled, any scanning, even while mounted as read-only will update the atime clock.

* <b>NFS</b>: The Probe host has be provided root-squash and read only access
    * For faster scanning, use an operating system that has nconenct support:
        * Ubuntu 20.04+
        * RHEL/Rocky 8.4+
* <b>Lustre</b>: The Probe host and container must be able to read as a root user
* <b>GPFS</b>: The Probe host and container must be able to read as a root user
* <b>SMB</b>: The Probe host should be mounted with a user in the BUILTIN\Backup Operators group to avoid file access issues. 
* <b>S3/Object</b>: We have tested internally with goofys as a method of imitating a filesystem
It is not recommend to scan anything in AWS Glacier or equivalent

## Hardware Requirement Examples

<b>Example A</b>: You have a server with 768GB of RAM:

* 154GB is for the Operating System, leaving 614GB of RAM...
* There are 100 million files to scan, that will occupy ~5GB of RAM, leaving 609GB of RAM...
    * 50-bytes per 'filename'
* This leaves 609GB of RAM available for the RAM index
```markdown
--ram-index-size-gb 609
```
  * This can scan up to 99TB of data using just RAM and no significant local SSD space is needed
      * This calculation is based on a 0.6% rule to accommodate similarity and deduplication hashes
* Use of a disk index you can scan far more data and the file count could exceed 10 billion with a 500GB file name cache

<b>Example B</b>: You have a server with 128GB of RAM and a Local SSD:

* 26GB is for the Operating System, leaving 102GB of RAM...
* There are 100 million files to scan, that will occupy ~5GB of RAM, leaving 97GB of RAM...
    * 50-bytes per 'filename'
* This leaves 97GB of RAM available for the RAM index
```markdown
--ram-index-size-gb 97
```
  * This can scan up to 15TB of data using just RAM and no significant local SSD space is needed
      * This calculation is based on a 0.6% rule to accommodate similarity and deduplication hashes
* Using a disk index you can scan far more data and the file count could be as high as 2 billion with a 100GB file name cache
    * 15TB of data requires 90GB of local SSD disk
    * 100TB of data requires 600GB of local SSD disk