# Sequence and genome searching

## Using cases

- Searching sequences against raws reads.
    - For checking existence:
        - Building database with raws reads and searching with query sequences,
          optionally using k-mer sketches for long query sequences.
    - For abundance estimation:
        - For long sequences (similar to taxonomic profiling):
            - Building database with query sequences, searching with raws reads, and profiling.
        - For short (shorter than reads length) sequences:
            - Not capable.
- Searching sequences against assemblies/genomes.
    - For checking existence:
        - For short sequences (short reads, e.g., detecting host contamination):
            - Building database with assemblies/genomes and searching with raws reads.
        - For long sequences:
            - Building database with assemblies/genomes using k-mer sketches.
    - For genome similarity estimation:
        - Building database with assemblies/genomes using k-mer sketches.
    - For taxonomic profiling:
        - For short reads:
            - Building database with assemblies/genomes, searching with raws reads, and profiling.
        - For long reads:
            - Split long reads into short reads before searching.

## Sequence search

KMCP can be used for fast sequence search against large scales of genomic datasets
as [BIGSI](https://github.com/Phelimb/BIGSI) and [COBS](https://github.com/bingmann/cobs) do.

KMCP reimplemented and modified the Compact Bit-Sliced Signature index (COBS) algorithm,
bringing a small database size and much faster searching speed
 (check the [benchmark](/kmcp/benchmark/searching)).

### Step 1. Building databases

The database building process is similar to that of metagenomic profiling,
with one difference:

1. Reference genomes are not splitted into chunks.

Taken GTDB for example:

    # mask low-complexity region (optional)
    mkdir -p gtdb.masked
    find gtdb/ -name "*.fna.gz" \
        | rush 'dustmasker -in <(zcat {}) -outfmt fasta \
            | sed -e "/^>/!s/[a-z]/n/g" \
            | gzip -c > gtdb.masked/{%}'

    # compute k-mers
    #   sequences containing "plasmid" in name are ignored,
    #   k = 21
    kmcp compute -I gtdb.masked/ -k 21 -B plasmid -O gtdb-r202-k21

    # build database
    #   number of index files: 32, for server with >= 32 CPU cores
    #   bloom filter parameter:
    #     number of hash function: 1
    #     false positive rate: 0.3
    kmcp index -j 32 -I gtdb-r202-k21 -O gtdb-r202-k21 -n 1 -f 0.3
    
    # cp name mapping file to database directory
    cp name.map gtdb-r202-k21.kmcp/

The size of database is 56GB. 

### Step 2. Searching

By default, `kmcp search` loads the whole database into main memory (RAM) for fast searching.
Optionally, the flag `--low-mem` can be set to avoid loading the whole database,
while it's much slower, >10X slower on SSD and should be much slower on HDD disks.
**Please switch on this flag when searching on computer clusters,
where the default mmap mode would be very slow for network-attached
storage (NAS)**

`kmcp search` supports FASTA/Q format from STDIN or files ([usage](/kmcp/usage/#search)).

    kmcp search -d gtdb.kmcp/ test.fq.gz -o result.tsv.gz

    22:21:26.017 [INFO] kmcp v0.8.0
    22:21:26.017 [INFO]   https://github.com/shenwei356/kmcp
    22:21:26.017 [INFO] 
    22:21:26.017 [INFO] checking input files ...
    22:21:26.017 [INFO]   1 input file(s) given
    22:21:26.018 [INFO] loading database with mmap enabled ...
    22:21:26.018 [INFO]   number of extra workers for every index file: 4
    22:21:26.328 [INFO] database loaded: gtdb.kmcp/
    22:21:26.328 [INFO] 
    22:21:26.328 [INFO] -------------------- [main parameters] --------------------
    22:21:26.328 [INFO]   minimum    query length: 30
    22:21:26.328 [INFO]   minimum  matched k-mers: 10
    22:21:26.328 [INFO]   minimum  query coverage: 0.550000
    22:21:26.328 [INFO]   minimum target coverage: 0.000000
    22:21:26.328 [INFO]   minimum target coverage: 0.000000
    22:21:26.328 [INFO] -------------------- [main parameters] --------------------
    22:21:26.328 [INFO] 
    22:21:26.328 [INFO] searching ...
    22:21:26.328 [INFO] reading sequence file: test.fq.gz

    22:21:26.376 [INFO] 
    22:21:26.376 [INFO] processed queries: 10, speed: 0.012 million queries per minute
    22:21:26.376 [INFO] 90.0000% (9/10) queries matched
    22:21:26.376 [INFO] done searching
    22:21:26.423 [INFO] 
    22:21:26.423 [INFO] elapsed time: 405.849288ms
    22:21:26.423 [INFO]

The result is tab-delimited.

|#query|qLen|qKmers|FPR       |hits|target         |chunkIdx|chunks|tLen   |kSize|mKmers|qCov  |tCov  |jacc  |queryIdx|
|:-----|:---|:-----|:---------|:---|:--------------|:-------|:-----|:------|:----|:-----|:-----|:-----|:-----|:-------|
|S0R0/1|150 |130   |3.1777e-06|2   |GCF_002872255.1|5       |10    |2583551|21   |87    |0.6692|0.0003|0.0003|0       |
|S0R0/1|150 |130   |1.1936e-03|2   |GCF_001434585.1|6       |10    |2221511|21   |74    |0.5692|0.0003|0.0003|0       |
|S0R0/2|150 |130   |6.2145e-07|4   |GCF_002872255.1|5       |10    |2583551|21   |90    |0.6923|0.0004|0.0004|1       |
|S0R0/2|150 |130   |3.1777e-06|4   |GCF_001434585.1|6       |10    |2221511|21   |87    |0.6692|0.0004|0.0004|1       |
|S0R0/2|150 |130   |2.4002e-05|4   |GCF_001438655.1|5       |10    |2038820|21   |83    |0.6385|0.0004|0.0004|1       |

Reference IDs (column `target`) can be optionally mapped to their names during searching:

    kmcp search -d gtdb-r202-k21.kmcp/ -N name.map test.fq.gz -o result.tsv.gz
    
Or after searching using `csvtk`:

    csvtk replace -t -C $ -f target -k name.map -p '(.+)' -r '{kv}' result.tsv.gz

Print the main columns only:

    csvtk head -t -C $ -n 5 result.tsv.gz \
        | csvtk rename -t -C $ -f 1 -n query \
        | csvtk cut -t -f query,FPR,qCov,target \
        | csvtk csv2md -t 

|query |FPR       |qCov  |target         |
|:-----|:---------|:-----|:--------------|
|S0R0/1|3.1777e-06|0.6692|GCF_002872255.1|
|S0R0/1|1.1936e-03|0.5692|GCF_001434585.1|
|S0R0/2|6.2145e-07|0.6923|GCF_002872255.1|
|S0R0/2|3.1777e-06|0.6692|GCF_001434585.1|
|S0R0/2|2.4002e-05|0.6385|GCF_001438655.1|

## Genome similarity estimation

KMCP can be used for **fast similarity estimation of newly assembled genome against known reference genomes**.
BLAST is an option while it focuses on local similarity.
Besides, the database is big and the searching speed is relative slow.

Genome sketching is a method of utilizing small and approximate summaries of
genomic data for fast searching and comparison.
[Mash](https://github.com/marbl/Mash) and [Sourmash](https://github.com/sourmash-bio/sourmash)
provide fast genome distance estimation using MinHash (Mash) or FracMinHash (Scaled MinHash) (Sourmash).
Here KMCP utilizes multiple sketches 
([Minimizer](https://academic.oup.com/bioinformatics/article/20/18/3363/202143), 
[FracMinHash](https://www.biorxiv.org/content/10.1101/2022.01.11.475838v2)
(previously named [Scaled MinHash](https://f1000research.com/articles/8-1006)) and
[Syncmers](https://peerj.com/articles/10805/)) for genome similarity estimation.

[Prebuilt databases](/kmcp/database) are available and users can also build custom databases following steps below.

### Step 1. Building databases

The database building process is similar to that of metagenomic profiling,
with few differences:

1. K-mer sketches instead of all k-mers are computed.
2. Reference genomes are not splitted into chunks.
3. Smaller false positive rate (`-f`) and more than one hash functions (`-n`) are used to improve accuracy.
4. Using a bigger k-mer size: 31.

Supported k-mer (sketches) types:

1. K-mer:
    - ntHash of k-mer (`-k`)
2. K-mer sketchs (all using ntHash):
    - FracMinHash    (`-k -D`), previously named Scaled MinHash
    - Minimizer      (`-k -W`), optionally scaling/down-sampling (`-D`)
    - Closed Syncmer (`-k -S`), optionally scaling/down-sampling (`-D`)

Taken GTDB for example:
    
    # compute FracMinHash (Scaled MinHash) with scale 1000
    #   sequence containing "plasmid" in name are ignored,
    #   k = 31
    kmcp compute -I gtdb/ -k 31 -D 1000 -B plasmid -O gtdb-r202-minhash

    # build database
    #   number of index files: 8, for server with >= 8 CPU cores
    #   bloom filter parameter:
    #     number of hash function: 3
    #     false positive rate: 0.001
    kmcp index -j 8 -I gtdb-r202-minhash -O gtdb.minhash.kmcp -n 3 -f 0.001
    
    # cp name mapping file to database directory
    cp taxid.map name.map gtdb.minhash.kmcp/

Attention:

- For small genomes like viruses, sketching parameters should be adjusted. 
For examples, setting a smaller scale like 10.
    
### Step 2. Searching

The searching process is simple and [very fast](https://bioinf.shenwei.me/kmcp/benchmark) (<1 second).

    kmcp search --query-whole-file -d gtdb.minhash.kmcp/ \
        --query-whole-file --sort-by jacc --min-query-cov 0.2 \
        --query-id genome1 contigs.fasta -o result.tsv

The output is in tab-delimited format:

```
 1. query,    Identifier of the query sequence
 2. qLen,     Query length
 3. qKmers,   K-mer number of the query sequence
 4. FPR,      False positive rate of the match
 5. hits,     Number of matches
 6. target,   Identifier of the target sequence
 7. chunkIdx, Index of reference chunk
 8. chunks,   Number of reference chunks
 9. tLen,     Reference length
10. kSize,    K-mer size
11. mKmers,   Number of matched k-mers
12. qCov,     Query coverage,  equals to: mKmers / qKmers
13. tCov,     Target coverage, equals to: mKmers / K-mer number of reference chunk
14. jacc,     Jaccard index
15. queryIdx, Index of query sequence, only for merging
```

A full search result:

|#query  |qLen   |qKmers|FPR        |hits|target         |chunkIdx|chunks|tLen   |kSize|mKmers|qCov  |tCov  |jacc  |queryIdx|
|:-------|:------|:-----|:----------|:---|:--------------|:-------|:-----|:------|:----|:-----|:-----|:-----|:-----|:-------|
|genome1 |9488952|18737 |0.0000e+00 |2   |GCF_000742135.1|0       |1     |5545784|31   |8037  |0.4289|0.7365|0.3719|0       |
|genome1 |9488952|18737 |3.1964e-183|2   |GCF_000392875.1|0       |1     |2881400|31   |3985  |0.2127|0.7062|0.1954|0       |

Reference IDs can be optionally mapped to their names, let's print the main columns only:

    kmcp search --query-whole-file -d gtdb.minhash.kmcp/ \
        --name-map name.map \
        --query-whole-file --sort-by jacc --min-query-cov 0.2 \
        --query-id genome1 contigs.fasta \
        | csvtk rename -t -C $ -f 1 -n query \
        | csvtk cut -t -f query,jacc,target \
        > result.tsv
    
|query   |jacc  |target                                                                                          |
|:-------|:-----|:-----------------------------------------------------------------------------------------------|
|genome1 |0.3719|NZ_KN046818.1 Klebsiella pneumoniae strain ATCC 13883 scaffold1, whole genome shotgun sequence  |
|genome1 |0.1954|NZ_KB944588.1 Enterococcus faecalis ATCC 19433 acAqW-supercont1.1, whole genome shotgun sequence|

Using closed syncmer:

    kmcp search --query-whole-file -d gtdb.syncmer.kmcp/ \
        --name-map name.map \
        --query-whole-file --sort-by jacc --min-query-cov 0.2 \
        --query-id genome1 contigs.fasta \
        | csvtk rename -t -C $ -f 1 -n query \
        | csvtk cut -t -f query,jacc,target

|query   |jacc  |target                                                                                          |
|:-------|:-----|:-----------------------------------------------------------------------------------------------|
|genome1 |0.3712|NZ_KN046818.1 Klebsiella pneumoniae strain ATCC 13883 scaffold1, whole genome shotgun sequence  |
|genome1 |0.1974|NZ_KB944588.1 Enterococcus faecalis ATCC 19433 acAqW-supercont1.1, whole genome shotgun sequence|
