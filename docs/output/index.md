# Understanding Output Overview
Periodically while running and at the end of a run, the probe will output data reduction results to the probe log file. These results are very helpful for understanding the data reduction that is expected when the data is placed on HPE GreenLake for File Storage as well as for helping to understand why that level of data reduction was achieved.


## Output format
The output will look something like this:
```text

--------------------------------current-probe-stats--------------------------------
Probe version: probe-version-4-4-703050
Scanned: 258.14GB out of 258.13GB (100.00%)
Files Scanned: 22481 files out of 22481 files (100.00%)

=============
Main Results:
=============
Total Global Data Reduction Factor = 5.32:1 (81.20% reduction)
Sparse Size = 258.14GB
Reduced Size = 48.54GB
Number of Inaccessible Files = 3 out of 22481 files (0.01% of scan)
Size of Inaccessible Files = 0.00B out of 258.13GB (0.00% of scan)
-
Duplicate Block Elimination Gain: 0.61% (1.56GB)
Zero Block Elimination Gain: 0.00% (1.80MB)
Number of Duplicate Chunks: 58917
Number of Zero Chunks: 35
-
Similarity Reduction Global DAC vs. Local DAC Gain: 1.69% (4.37GB out of total bytes using Similarity: 233.62GB)
Number of Similar Chunks: 4572414 out of 5414721 total unique chunks
Average Chunk Size: 49.99KB
Similarity Percentage: 84.44%
Average Size of Chunks Using Similarity: 53.58KB
Average Gain post DAC Per Similarity Match: 1.00KB
Vast Array Performance Impact: green
-
Local Compression Gain including DAC: 79.03% (204.01GB out of a total Compression scan of 252.21GB)
Compression ratio for local compress only: 4.88:1

==================
Adaptive Chunking:
==================
...

=======================
Data Aware Compression:
=======================
...

======================
Experimental Features:
======================
...
```
There are two types of output above: normal or routine information relevant to most, and more advanced information that is more internal in nature (shown here with ...). In this article we will consider both types of information in the output, but please focus on the routine information as that is almost always more relevant.

## Routine Considerations (Main Results)
The intent of this output is to summarize what the probe has found so far. The interesting results are:

* `Scanned` shows the space before reduction
* `Files Scanned` shows the number of files in the entire data set that were scanned
* `Total Global Data Reduction Factor` shows how effectively data reduction was done overall. This value includes compression, deduplication, and similarity reduction.  
    * `Reduced Size` is the space after reduction
    * `Sparse Size` should be ignored unless the probe is run with `--sparse-mode` as described below.
    * `Number/Size of Inaccessible Files` indicates data the probe tried to read but couldn't. If this number is large, the probe results are not valid. This almost always happens due to permission issues or files being deleted while the probe was running.
* `Duplicate Block Elimination Gain` shows how much space is saved just by removal of duplicate blocks.
    * `Number of Duplicate Chunks` shows literally how many blocks were identical to other existing blocks.
    * `Zero Block Elimination Gain` tells you how much of the gain from deduplication was due to zero blocks. That helpful for understanding the implications of the next item.
    * `Number of Zero Chunks` is a count of number of chunks that are all zeros. That often indicates sparse files. If the number of such chunks is high relative to the number of chunks (exceeding say 10%), the probe estimates may be misleading. Use tools such as du and df to determine the actual space used and compare that to the probe's report of the space scanned. If there is a large difference, sparse files are likely to blame. If your file system supports the advanced ioctl for sparse file reporting (Lustre and XFS do), you can try running the probe again with `--sparse-mode`.
* `Similarity Reduction Global DAC vs. Local DAC Gain` is the gain from similarity with data aware compression vs. the gain without similarity. This is just a more verbose way of saying "this is how much gain similarity provided."
    * `Number of Similar Chunks / Similarity Percentage` is the number of data chunks that benefited from similarity matching. The percentage is simply the number of chunks that benefited from similarity divided by the total number of chunks. A high value for the similarity match percentage (significantly over 10%) and a low value of `Average Gain Post DAC Per Similarity Match` relative to `Average Size of Chunks Using Similarity` is a potential problem. This indicates a high similarity match rate, but a low gain from those matches. The amount reported is bytes per chunk.
    * `Average Chunk Size` is the average size (before reduction) of all chunks
    * `Average Size of Chunks Using Similarity` is the average size (before reduction) of a chunk that benefited from similarity
    * `Array Performance Impact` should be ignored for now.  
* `Local Compression Gain` shows how much space would be saved just by transparent compression as files are saved. This is also helpfully expressed at the end via Compression ratio for local compress only. Essentially that ratio vs. the reported Total Global Data Reduction factor shows how much better DRR was thanks to global deduplication and similarity reduction.


In the above example we can see that we scanned 22481 files that consumed 258GB of space before any data reduction. After data reduction the probe predicts the files will consume 48GB of space for a reduction of 81%. Of that simple compression gains 79% (204GB), deduplication 1% (1GB), and similarity 2% (4GB). Please keep in mind these aren't typical results as actual data reduction varies widely for different data sets.

## Advanced Considerations
In addition to the common and most relevant output described above, there are more advanced bits of information shared by the probe. Most of this information is only relevant to VAST engineering (we hope you can share it with us) but we document it here for the curious.

 

Here is an example of the more advanced outputs:
```text
==================
Adaptive Chunking:
==================
min_chunk_size=16384 max_chunk_size=65043 desired_chunk_size=29950 inverse_probability=13999 split_threshold=17871601040105585914
Theoretical Average Chunk Size: 29.25KB (error: -70.92%)
Number of chunks split via hash: 2423353 (44.75%)
Number of chunks split via buffer end: 44620 (0.82%)
Number of chunks split via max size reached: 2969226 (54.84%)

=======================
Data Aware Compression:
=======================
Total Number of Predictions: 5414686
Predictions Per Encoder Type: {ENCODER_NONE=5402314, ENCODER_SHUFFLE=11164, ENCODER_DELTA_ENCODE=681, ENCODER_DELTA_ENCODE_4_SHUFFLE=527}
Percentage of Chunks Per Encoder:
- Encoder ENCODER_NONE: 99.77%
- Encoder ENCODER_SHUFFLE: 0.21%
- Encoder ENCODER_DELTA_ENCODE: 0.01%
- Encoder ENCODER_DELTA_ENCODE_4_SHUFFLE: 0.01%

Encoding Sampling Reduction Summary (sampling 1.99%):
----------------------------------------------------------------------------------------------------------------------------------------------------
Encoders | None | Shuffle | Delta Shuffle | Delta
----------------------------------------------------------------------------------------------------------------------------------------------------
DRR (Global) | 5.33 | 3.60 | 3.07 | 4.83
Compressed Size | 48.44GB | 71.73GB | 84.06GB | 53.44GB
Num Chunks Improved Percentage | 98.87% | 13.43% | 13.25% | 13.36%
Num Chunks Improved | 5353433 | 727142 | 717668 | 723182
Total Chunks Num | 5414721 | 5414721 | 5414721 | 5414721
Similarity Reduction Percentage | 1.68% | 1.77% | 2.62% | 2.01%
Similarity Reduction | 4.34GB | 4.59GB | 6.78GB | 5.19GB
Total Bytes Using Similarity | 233.62GB | 233.62GB | 233.62GB | 233.62GB
Similarity Reduction Gain if ref chain Percentage | 86.88% | 0.00% | 0.00% | 0.00%
Similarity Reduction Gain if ref chain | 224.86GB | 0.00B | 0.00B | 0.00B

Data Aware Compression Accuracy:
Total Chunks Compared for Discovering Optimal Encoding: 108046
Total Correct Optimal Encoding Predictions: 107781
Total Wrong Optimal Encoding Predictions: 265
Correct Predictions Percentage: 99.75%
Predictions Per Encoder Type: {ENCODER_NONE=107825, ENCODER_SHUFFLE=195, ENCODER_DELTA_ENCODE=14, ENCODER_DELTA_ENCODE_4_SHUFFLE=12}
Wrong Predictions Per Encoder Type: {ENCODER_NONE=98, ENCODER_SHUFFLE=160, ENCODER_DELTA_ENCODE=1, ENCODER_DELTA_ENCODE_4_SHUFFLE=6}
Wrong Predictions Percentage Per Encoder:
- Encoder ENCODER_NONE = 0.09%
- Encoder ENCODER_SHUFFLE = 82.05%
- Encoder ENCODER_DELTA_ENCODE = 7.14%
- Encoder ENCODER_DELTA_ENCODE_4_SHUFFLE = 50.00%
* Note: Wrong predictions does not mean that there is no gain from the encoder, but rather that there is a better one.

Total Pre-Encoding Compressed Size of Chunks Used in Predictions: 1.07GB
Total Post-Encoding Compression Size of Chunks Used in Predictions: 1.07GB
Total Optimal Compression Size of Chunks Used in Predictions: 1.07GB
Total Size Difference Between Predicted and Optimal Encoded Compression: 274.36KB (Optimal compression size is smaller than the predicted compression size by 0.02%)
Approximate Total Local Data-Reduction Factor Without Data Aware Compression: 4.83:1 (79.30% reduction)
Actual Total Global Data-Reduction Factor Without Data Aware Compression (available at 100% sampling): N/a

======================
Experimental Features:
======================
Similarity Reduction Gain if ref chain: 1.74% (4.49GB out of total bytes using Similarity: 233.62GB)
Extra space gain in optimal compression: 47.57GB
-
Extra local compression space gain in case of using compression_level 8: 4.35GB
-
Extra local compression space gain in case of using compression_level 8: 3.16GB
```

* ** Adaptive Chunking ** 
    * `min_chunk_size=AAA max_chunk_size=BBB desired_chunk_size=CCC` are all internal settings that we may change from probe version to probe version. Otherwise they should be ignored.
    * `Theoretical Average Chunk Size` should be ignored
    * `Number of chunks split via XXXX`: adaptive chunking automatically adjusts the size of data chunks to improve deduplication and similarity matching. These three metrics tell us a bit about how we are doing.
        * ** via hash**: the count of chunks that were split using the automated data sensitive splitting. Typically this will be a high value.
        * ** via buffer end**: the count of chunks that were split simply because we reached the end of the relevant data stream. A likely cause is simply the end of a file.
        * ** via max size reached**: the count of chunks that were split because the chunks would have otherwise been too large.
* ** Data Aware Compression ** 
    * ** Encoding Sampling Reduction Summary ** summarizes the various different data aware compression (DAC) encodings and how well they worked for all of the data chunks. The probe randomly selects some number of chunks (sampling) and tries all encoding schemes. This is not what VAST or the probe does for all chunks as it is too expensive. Instead, the system examines a bit of each data chunk and decides on the DAC encoding scheme to use and then uses it - we call this prediction. This table show how the different schemes fared and helps us understand if our predictions are accurate. In general this table can be ignored.
    * ** Correct Predictions Percentage ** tells us how often our predictions where correct. This calculation is based upon these values:
        * ** Total Chunks Compared for Discovering Optimal Encoding**: how many chunks were sampled for checking purposes
        * ** Total Correct Optimal Encoding Predictions**: how often the predictor was correct
        * ** Total Wrong Optimal Encoding Predictions**: how often the predictor was wrong
    * ** Total Size Difference Between Predicted and Optimal Encoded Compression ** indicates how well our predictor selected the optimal DAC encoding scheme in terms of space used. If the number here is small (less than 5%) then the predictor is doing well. If it is larger, please let us know. These are the inputs to this calculation:
        * ** Total Pre-Encoding Compressed Size of Chunks Used in Predictions **: size of chunks before reduction
        * ** Total Post-Encoding Compression Size of Chunks Used in Predictions **: size of chunks after reduction
        * Total Optimal Compression Size of Chunks Used in Predictions - the optimal reduction (basically trying all possible encodings based upon sampling)
    * Approximate Total Local Data-Reduction Factor Without Data Aware Compression - our estimate (based upon sampling) of the data reduction without DAC. Basically if the value here is smaller than the value reported in the first part of the summary, DAC was a win.
* Experimental Features
    * Extra space gain in optimal compression - this considers advanced data reduction algorithms that are under consider for future versions but have not yet implemented in actual released products. If you see a very large value here relative to the total data, let us know. That's very interesting to us! 
    * Extra local compression space gain in case of using compression_level 8 - this indicates how much space could be saved in local compression if the most expensive ZSTD compression setting. This isn't done on real clusters as it impact performance, but it's a useful metric for our engineering. Typically the additional savings is minimal which is good.
 