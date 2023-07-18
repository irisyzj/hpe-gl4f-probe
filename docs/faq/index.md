# General FAQ

**Q: How does the probe handle symbolic links?**<br>
**A**: The probe ignores symbolic links. Thus if it is scanning a directory tree and encounters a symbolic link to some other area in the file system, it will not follow it.

**Q: How does the probe handle hard links?**<br>
**A**: The probe attempts to detect if two files in the tree it is scanning point to the same data and automatically ignores the duplication.

**Q: How does the probe handle sparse files?**<br>
**A**: By default the probe is not aware of sparse files. This means that it will read zero values for the sparse regions of the files, which can result in artificially high data reduction. The probe reports zero chunks to hint at this potential issue. Refer to Understanding VAST Probe Output for more details. Note that the probe can be run to recognize sparse files on some files systems as described in the document just referenced.

**Q: Can the probe scan multiple unrelated directory trees?**<br>
**A**: Yes it can. This is done by providing multiple --input-dir values.

# Security FAQ

The HPE GreenLake for File Storage Data Reduction Estimation Probe software is provided at zero cost with zero warranty to HPE’s current and prospective customers in order to accurately estimate Data Reduction Rates of specific data not yet on HPE Storage systems. The probe software is run on physical or virtualized customer-maintained hardware and analyzes data that the customer allows access to through traditional filesystem based access. The results of the probe are used to determine a Data Reduction Rate which will often be used to project an aggregate financial savings for HPE’s current and prospective customers.

**Q: Where does the VAST Probe originate?**<br>
**A:** The HPE GreenLake for File Storage Data Reduction Estimation Probe is a Docker container of scripts and libraries maintained and assembled solely by HPE and VAST Data engineering which is updated frequently, usually quarterly. The links to download the probe are posted on this GitHub repository.

**Q: Where does the VAST Probe run?**<br>
**A:** The HPE GreenLake for File Storage Data Reduction Estimation Probe is designed to be run within a customer environment on physical or virtualized customer-maintained equipment. The provided container requires a base Linux operating system which is expected to be installed and updated by the customer before the probe is launched.

**Q: What information does the VAST Probe collect?**<br>
**A:** The HPE GreenLake for File Storage Data Reduction Estimation Probe generates a series of logs for each iteration of data scanning. These logs are by default saved on the same physical or virtualized customer-maintained equipment that the probe runs. These logs contain references to paths which have been provided as inputs, and can refer to any path within that directory structure when making declarative statements about data reduction results. The analysis log file that is generated upon completion of the Data Reduction Probe prints each full path with figures about data reduction rate for that path. In addition, a secondary section of same analysis log file prints aggregate information about specific file extensions with figures about data reduction rate for that file extension.

**Q: What information does the HPE GreenLake for File Storage Data Reduction Estimation Probe send back to HPE?**<br>
**A:** The HPE GreenLake for File Storage Data Reduction Estimation Probe as built-in call home telemetry which is on by default when executed assuming the probe has access to specific HPE endpoints via the internet. While the probe is running, telemetry logs will be sent approximately every 5 minutes. These telemetry logs, by default, omit references to full paths with the exception of the of the root input path and simply upload a percentage-based status of the probe as well as any error messages. The final telemetry log is similar to the local analysis log file but, by default, removes full paths with the exception of the of the root input path. The final telemetry log will send the aggregated data reduction rates based on file extensions as illustrated below:
```text
file extension statistics:
file type .xlsx, original_size=143.7GB, global_compression_reduced_size=126.6GB, global_compression_factor=1.14, dedup_percentage=10.34%, similarity_match_percentage=15.12%, similarity_gain=310.9MB, local_compression_only_size=126.9GB
file type .tsv, original_size=291.5GB, global_compression_reduced_size=30.8GB, global_compression_factor=9.47, dedup_percentage=1.95%, similarity_match_percentage=84.83%, similarity_gain=9.6GB, local_compression_only_size=40.4GB
```

**Q: Who can access the logs sent to VAST Data?**<br>
**A:** Anyone at HPE engineering or sales has access to the call home backend that is used as the telemetry destination for the HPE GreenLake for File Storage Data Reduction Estimation Probe.

**Q: What actions are performed with the logs sent to HPE?**<br>
**A:** The telemetry logs are primarily used by sales to determine a Data Reduction Rate which will often be used to project an aggregate financial savings for HPE’s current and prospective customers. Alternatively, any telemetry logs can be used to determine an expected Data Reduction Rate for a given industry or use case which may be similar to a sales team’s customer which has not run the probe. HPE engineering also uses the telemetry data for bug fixes and over all improvements to the software and user experience.

**Q: How do I control what the VAST Probe sends back to VAST Data?**<br>
**A:** This call home telemetry feature can be disabled at runtime with the added flag:
```bash
--dont-send-logs
```
If you wish to send file names with the default telemetry logs, add the following flag:
```bash
--send-logs-with-file-names
```


