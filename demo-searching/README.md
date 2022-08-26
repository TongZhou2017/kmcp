# Demo of sequence/genome searching

## Dataset

### References

    csvtk join -t -f id \
            <(seqkit stats -j 10 refs/*.fasta.gz -T -b \
                | csvtk mutate -t -n id -p "(.+)\.fasta") \
            <(csvtk add-header -t -n id,name name.map) \
        | csvtk cut -t -f file,format,type,num_seqs,sum_len,name \
        | csvtk csv2md -t

file                  |format|type|num_seqs|sum_len|name
:---------------------|:-----|:---|:-------|:------|:----------------------------------------
NC_000913.3.fasta.gz  |FASTA |DNA |1       |4641652|Escherichia coli str. K-12 substr. MG1655
NC_002695.2.fasta.gz  |FASTA |DNA |1       |5498578|Escherichia coli O157:H7 str. Sakai
NC_010655.1.fasta.gz  |FASTA |DNA |1       |2664102|Akkermansia muciniphila ATCC BAA-835
NC_011750.1.fasta.gz  |FASTA |DNA |1       |5132068|Escherichia coli IAI39
NC_012971.2.fasta.gz  |FASTA |DNA |1       |4558953|Escherichia coli BL21(DE3)
NC_013654.1.fasta.gz  |FASTA |DNA |1       |4717338|Escherichia coli SE15
NC_018658.1.fasta.gz  |FASTA |DNA |1       |5273097|Escherichia coli O104:H4 str. 2011C-3493
NZ_CP007592.1.fasta.gz|FASTA |DNA |1       |5104557|Escherichia coli O157:H16 strain Santai
NZ_CP028116.1.fasta.gz|FASTA |DNA |1       |5648177|Escherichia coli O26 str. RM8426

## Genome similarity estimation

### Syncmer

    # computing k-mers
    kmcp compute \
        --in-dir refs/ \
        --seq-name-filter "plasmid" \
        --kmer 31 \
        --syncmer-s 15 \
        --scale 62 \
        --out-dir refs-k31-syncmer \
        --force

    # indexing k-mers
    kmcp index \
        --in-dir refs-k31-syncmer/\
        --num-hash 3 \
        --false-positive-rate 0.01 \
        --out-dir refs-k31-syncmer.kmcp \
        --force

    # searching
    kmcp search \
            --db-dir refs-k31-syncmer.kmcp \
            --query-whole-file \
            --min-query-cov 0.5 \
            --keep-top-scores 0 \
            --sort-by jacc \
            --name-map name.map \
            refs/NC_018658.1.fasta.gz \
        | csvtk cut -t -C '$' -f '#query,target,qCov,tCov,jacc' \
        | csvtk csv2md -t -C '$'

|#query     |target                                   |qCov  |tCov  |jacc  |
|:----------|:----------------------------------------|:-----|:-----|:-----|
|NC_018658.1|Escherichia coli O104:H4 str. 2011C-3493 |1.0000|1.0000|1.0000|
|NC_018658.1|Escherichia coli O26 str. RM8426         |0.7439|0.7189|0.5763|
|NC_018658.1|Escherichia coli str. K-12 substr. MG1655|0.6041|0.6768|0.4688|
|NC_018658.1|Escherichia coli BL21(DE3)               |0.5972|0.6807|0.4665|
|NC_018658.1|Escherichia coli O157:H16 strain Santai  |0.5782|0.5868|0.4109|
|NC_018658.1|Escherichia coli O157:H7 str. Sakai      |0.5482|0.5322|0.3699|

### FracMinHash (Scaled MinHash)

    # computing k-mers
    kmcp compute \
        --in-dir refs/ \
        --seq-name-filter "plasmid" \
        --kmer 31 \
        --scale 1000 \
        --out-dir refs-k31-minhash \
        --force

    # indexing k-mers
    kmcp index \
        --in-dir refs-k31-minhash/\
        --num-hash 3 \
        --false-positive-rate 0.01 \
        --out-dir refs-k31-minhash.kmcp \
        --force

    # searching
    kmcp search \
            --db-dir refs-k31-minhash.kmcp \
            --query-whole-file \
            --min-query-cov 0.5 \
            --keep-top-scores 0 \
            --sort-by jacc \
            --name-map name.map \
            refs/NC_018658.1.fasta.gz \
        | csvtk cut -t -C '$' -f '#query,target,qCov,tCov,jacc' \
        | csvtk csv2md -t -C '$'

|#query     |target                                   |qCov  |tCov  |jacc  |
|:----------|:----------------------------------------|:-----|:-----|:-----|
|NC_018658.1|Escherichia coli O104:H4 str. 2011C-3493 |1.0000|1.0000|1.0000|
|NC_018658.1|Escherichia coli O26 str. RM8426         |0.7499|0.7234|0.5828|
|NC_018658.1|Escherichia coli str. K-12 substr. MG1655|0.6064|0.6833|0.4734|
|NC_018658.1|Escherichia coli BL21(DE3)               |0.5965|0.6893|0.4701|
|NC_018658.1|Escherichia coli O157:H16 strain Santai  |0.5852|0.5958|0.4189|
|NC_018658.1|Escherichia coli O157:H7 str. Sakai      |0.5527|0.5383|0.3750
