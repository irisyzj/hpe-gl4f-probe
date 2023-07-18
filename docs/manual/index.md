# Manual Execution Overview

The HPE GreenLake for File Storage Data Reduction Estimation Probe is a long running process in a docker container. The docker container needs to run on a linux system that has read only access to the files you want to examine for data reduction as well as reasonable memory and substantial fast local disk.

When having issues with the probe_launcher.py script or you need more experimental features, you should use this page.

[TOC]

## Manual Execution Procedure

Follow the steps in [Prerequisites](../prerequisites/index.md) to verify requirements are met to properly run the probe.  Get docker container image via links in [Deployment](../deployment/index.md).

When the probe docker container is launched you'll then be able to connect into it and then run the probe itself with key configuration information. The probe will then run until completion and report results. 

### Download the bundle

1. Download the docker image
    Follow the download links in [Deployment](../deployment/index.md) to download the bundle.
    Then set the variable for the build number:

    `export PROBE_BUILD=[PROBE BUILD NUMBER]`


2. Untar the bundle:

    `tar -xzf ${PROBE_BUILD}.probe.bundle.tar.gz`


3. Load the docker image:

    `docker load -i ${PROBE_BUILD}.probe.image.gz`

    This step will take a few minutes without meaningful output.

Tag the loaded image by doing a docker images and noting the new image, and it's ID.  Recall that images are identified by unique image IDs and human readable tags. Tag it as shown below - get the ID from the docker images output, and the value of the name is by convention the probe build.
```bash
docker images
```
Notice the Image IDs in the output list
```bash
docker tag <hex image tag value from output> vast-probe-${PROBE_BUILD}
```

### Configure the run

Launch a 'screen' session (or tmux). We recommend some kind of long lived session tool since the probe can take a very long time to run and we do not want it to terminate if there is an issue with the client system.
```bash
screen -R probe
```
Run the container while mapping the required directories 
run with the image tag/name you set earlier. The `-v` specifies mounts from the real operating system that should be made available to docker. These are directories that the probe can use and scan. Include as many `-v`'s as needed, just ensuring that at least one is the actual probe scratch directory (`/mnt/probe` in this case).

```bash
docker run -v /mnt/fileserver1:/mnt/fileserver1 -v /mnt/probe:/mnt/probe -it vast-probe-${PROBE_BUILD}
from within the docker container, Create relevant output directories, eg:
sudo mkdir -p /mnt/probe/vast-probe/output
sudo mkdir -p /mnt/probe/vast-probe/db
sudo chmod -R 777 /mnt/probe/vast-probe
#note: If you get permission denied then disable selinux on your host
Edit probe config file:
vim /vast/install/probe/sim_init_file.yml
```
 

See example config below, but also some items to note:

* `input_dir`:  you can specify more than one. Just prefix a newline with '-'.  This will allow the probe to scan multiple mountpoints/filesystems. Each input directory is scanned in a parallel thread which can slightly improve probe scan times.
* `output_dir` : this is where the summary files and some stats files will go. this is relatively small (< GB, although could get larger if you are scanning a lot of paths)
* `metadata_dir` : if using disk based indexes the space here needs to be pretty large (1% of total dataset to be very safe).
* `match_disable`: if you set to '1' , it will do 'local-only' compression/dedup.  This completes much more quickly, but will not do any similarity hashing.
* `max_number_of_files`: This effectively pre-allocates some RAM to hold for file pointers.  Set this value to somewhat higher than the total number of files you expect the probe to scan.  Every 1-million files takes up 50MB of RAM.  1-billion is 50GB.  Make sure not to set a value that causes the file pointer cache to exceed 50% of system RAM.
* `disk_size_gb`: set this to use disk based index.  If you set to 0 it will instead use a RAM based index (see next variable) Index is ~80% of the probe metadata so rule of the thumb here so if you have a dedicated SSD-based file system for probe md the rule of thumb is to put here 80% of the disk size. And remember the free disk space size needs to be 0.6% of the total dataset size (this has a safety margin).
* `ram_size_gb`: if disk index is not used the probe will use RAM for indexing. This is faster but may produce inaccurate results for large data sets. If this value is left unset the probe will use 80% of the available system memory.
* `IOPS_limit`: can be used to limit the read rate from the target system. The IO size is the chunk size (default 32K), e.g. IOPS_limit: 1000 → ~320 MB/sec. 

** Example config**:

```bash
input_dir:
  - '/mnt/fileserver/data/stuff'
filter: '*'
output_dir: '/mnt/probe/vast-probe/output' # dir for log files
metdata_dir: '/mnt/probe/vast-probe/db' # dir for probe metadata

regexp_filter: '' # files/directories matching the filter will NOT be scanned by the probe
 
send_from: 'andy@vastdata.com'
send_to:
 - 'andy@vastdata.com'
 - 'probe.callhome@vastdata.com'
 
remote_monitoring_freq: 100 # sending mail with stats line, every remote_monitoring_freq seconds
SMTP_host: 'localhost' # put an SMTP relay here
 
remove_db_dir: 0 #remove db dir after each run? 1 for yes, 0 for no
ignore_links: 1 #1 for yes, 0 for no
 
IOPS_limit: 0 #for no limit, put 0
number_of_threads: 0 # for one thread per core, put 0
printing_frequency: 1 # in seconds
open_files_limit: 0 #for no limit, put 0
obfuscate_files_names: 0 #1 for yes, 0 for no
match_disable: 0 #1 to disable matches, 0 to enable
ram_size_gb: 0 # RAM for indexes (in GB), 0 will make the probe us ~80% of the available system memory
disk_size_gb: 100 # if set will use disk to store the similarity index.
 
pause: #'7:15' #hh:mm or leave blank
resume: #'17:16' #hh:mm or leave blank
 
split:
...
```

Once you are satisfied, copy the .yml file to somewhere outside of the container (NFS mount or via SCP), since it will not survive container restart

```bash
cp /vast/install/probe/sim_init_file.yml /mnt/probe/
```

### Launch the probe run
While still connected to the probe's docker container, go to the probe's home directory.
Note: if you need to run the probe a second time you can copy the save `sim_init_file.yml` file from `/mnt/probe` into the container at `/vast/install/probe`. 

```bash
cd /vast/install/probe
```

Run it:
sudo is required if root is needed in order to access one of the directories configured in the init file. 

```bash
sudo python3 ./probe.py
```
## Probe Stages
Some of these stages run concurrently (eg: Treewalk can run in the background throughout)

### Treewalk Phase
When the probe is first kicked off, it  builds a list of all files, along with the size-in-bytes for each file.  This process has recently been parallelized to try and use more threads to perform this treeewalk, however depending on the source filesystem, this may still take a significant amount of time.   Note that this runs in the background, such that the probe can make progress with other stages even while the treewalk phase is active.

As an alternative, you can specify the `--csv` option to point to a CSV file which looks like this:

```bash
/path/to/file.file,1234
```
where 1234 = sizeInBytes


### DB Initialization Phase
The probe needs to initialize the Dictionary/Database which is used for storing matches.  Depending on the speed of the storage which is hosting the database (specified via 'metadata_dir' ) , this can take some time.  Also note that the 'disk_size_gb' parameter is directly related to how large the DB will be.  During this phase, the probe will pre-allocate the DB by writing XX-GB to the metadata_dir.  

### DataScan Phase
Once initialization has occurred, this is when the actual probe-scanning happens.  During this time, multiple threads are walking through the generated list of files and reading them to generate the various hashes which are then inserted into the DB.  You can monitor progress during this phase as described below.

 

## Monitoring progress
 

While the probe is running, there are 2 ways to get progress:

* Watch the screen

```bash
sudo python3 ./probe.py
mail sending off
Scanning input directories, this might take a while...
Scanned 144932 files, size 3.2TB
File scan completed
open file limit is 65536, it is recommended to allow as many open files as possible
Initializing probe.
336.386 GB/3.1718 TB (10.4%)     process_rate = 850.45 MB/sec   factor = 1.54
```

* Check the log file

```bash
tail -f /mnt/probe/vast-probe/output/probe_Mon_Jan_21_11_57_52_2019.log
```

The log will give you information like this:
```text
n_chunks = 482991, n_matched_chunks = 392628, dedups = 918, match_percent = 81.291% , sum_of_gain = 403245081, gain = 64.877, avg_gain_per_match = 1027.04, avg_match_hashes_per_match = 9.28467, decompressed_sum = 1.212 GB, compressed_sum = 204.86 MB, factor = 6.05886, ratio = 0.165048, sum_of_self_compress = 592.76 MB size_of_data_processed = 1.212 GB/1.324 GB 91.5786% number_of_inaccessible_files = 0/517401 size_of_inaccessible = 0 B/1.324 GB READ = 1.212 GB, RE-READ = 940.70 MB, Total READ = 2.131 GB process_rate = 34.33 MB/sec
```

## Understanding the ouput

The summary probe output is described in Un[derstanding Output](../output/index.md)

### Low Level Output
In addition to the previous output, the probe will also output lower level information periodically. These days that information is not typically useful, but here is an explanation just in case.

* `n_chunks` - amount of chunks processed by the probe (default size is 32K)
* `avg_chunk_size` - average chunk size
* `n_matched_chunks` - amount of chunks identified as similar to pre existing chunks by similarity search 
* `match_percent` - percentage of chunks identified as similar to pre existing chunks by similarity search 
* `sum_of_gain` - total space saved by similarity compression
* `gain` - percentage of space saved by similarity compression
* `avg_gain_per_match` - average amount of space saved per chunk from similarity compression
* `avg_match_hashes_per_match` - average amount of matching hashes found during similarity seach
* `n_duplicate_chunks` - amount of identical chunks found
* `dedup_percent` - percentage of identical chunks found
* `original_size` - amount of data processed by the probe
* `compressed_sum` - estimated size of data post compression, dedup and similarity compression
* `factor` - compression factor (original_size / compressed_sum)
* `ratio` - 1 / factor
* `sum_of_self_compress` - data size if only local compression (with the given chunk size) was applied
* `size_of_data_processed` - progress indication 
* `number_of_inaccessible_files` - number of files that were found in the initial scan but the probe didn't manage to read from when trying to process them
* `size_of_inaccessible` - amount of data that were found in the initial scan but the probe didn't manage to read from when trying to process them
* `READ` - amount of scanned data 
* `RE-READ` - amount of data that was re-read in order to perform global compression
* `Total READ` - READ + RE-READ
 

I thought it would be helpful to share results from a test run and an interpretation of those results for the benefit of others:

Here’s the last line of output with a summary:
```text
n_chunks = 1120059762, avg_chunk_size = 32754.2, n_matched_chunks = 507860164, match_percent = 45.3422% , sum_of_gain = 99.367 GB, gain = 0.603369, avg_gain_per_match = 210.087, avg_match_hashes_per_match = 3.4, n_duplicate_chunks = 529286922, dedup_percent=47.2552, original_size = 33.3664 TB, compressed_sum = 2.7112 TB, factor = 12.3069, ratio = 0.0812552, sum_of_self_compress = 16.0828 TB, size_of_data_processed = 33.3664 TB/33.7412 TB 98.8889%, number_of_inaccessible_files = 17233/880015, size_of_inaccessible = 384.025 GB/33.7412 TB, READ = 33.3664 TB, RE-READ = 15.1350 TB, Total READ = 48.5014 TB
```
 
And the definitions that I think are most pertinent:
* `match_percent` - percentage of chunks identified as similar to pre existing chunks by similarity search 
* `sum_of_gain` - total space saved by similarity compression
* `gain` - percentage of space saved by similarity compression
* `dedup_percent` - percentage of identical chunks found
* `original_size` - amount of data processed by the probe
* `compressed_sum` - estimated size of data post compression, dedup and similarity compression
* `factor` - compression factor (original_size / compressed_sum)
* `sum_of_self_compress` - data size if only local compression (with the given chunk size) was applied
* `size_of_data_processed` - progress indication 

This means the probe processed 33TB of data. The “native” compressed size would have been 16TB
the actual compressed size including compression, dedup, and similarity compression was 2.7TB, thus the total factor of savings was 12 (33/2.7).
 
Digging a little deeper we see that the majority of the savings came from dedup (47% of the chunks were identical) and compression as it looks like similarity compression saved 0.6% for a total of 99GB.


## Probe Analyze
After the probe completes a run it will automatically analyze its own output from the log files and generate an analysis log (still quite long) with a breakdown by directory and file extension of the data reduction achieved.

In rare cases you may need to run this manually, here's how:

```bash
cd /vast/install/probe
python3 ./probe.py --analyze_log <probe log file>
.... output about processing files ....
Processed 1967860 files             
Writing probe run analysis to ..../probe_Date.log.analysis
```

## I/O Behavior
* **Speed**: From a scan-speed perspective, what we've found is that on average we see approximately ~60 MByte/sec per physical CPU core when running the probe in full "similarity hash" mode (default value for `match_disable`).  Thus, a 20-core system would net approximately 1.2 GByte/sec. Having that said performance is also highly dependant on the disk latency of the target system being scanned and is often delayed by doing random reads on that system. 

* **Read amplification**: The way our similarity hasher works, if it discovers any matches, it will need to re-read a portion of the dataset again to look for additional opportunities for dataReduction.  In the case where your data has a lot of similarity, this can result in significant read-amplification.  Therefore, when determining the amount of time it will take to scan a file-system, it is necessary to allow the probe to run for a period of time to determine the approximate 'Re-Read' ratio.
    * look at the `/mnt/probe/db/*.stats` output to see.
    
* **match_disable=1**: If you choose this setting (non-default), the probe will bypass similarity hashing, and instead only look for local compression opportunity, and full-chunk matches (for dedup).  This is much less CPU intensive, and we've found that the bottleneck will typically be either networking or the filesystem which it is scanning, up to a point.  In my testing on a system with 25gigE, using this mode saw an average of 1.3GByte/sec (about 66MB/sec/physCore).  At times the network throughput got close to line-rate (2+GByte/sec).
    * If you have a subset of data which is representative of a larger set: it would be advisable to run against the smaller set in this mode first, to determine the local compression & dedup rates.  Once that rate is established, running the probe again in similarity-hash mode against the full dataset is recommended.
