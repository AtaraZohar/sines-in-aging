TODO document all stages

# Formats we use

[.fasta](https://en.wikipedia.org/wiki/FASTA_format) - a record header line, followed by AGCT characters.
But can contain other letters that represent reading uncertainty, e.g. Y = C, T or U.

[.fastq](https://en.wikipedia.org/wiki/FASTQ_format) - similar but also contains per-character *(Q)uality* info.

Our particular FASTQ inputs don't wrap lines, so contain exactly 4 lines per record.

.fastq.gz, .fastq.zst - FASTQ file, compressed with off-the-shelf compression tools (gzip, zstd).

We have a function `open_any()` that does de/compression automatically, depending on file extension.

# Install tools

On ubuntu linux, try running `./INSTALL/ALL.sh`.

Note it installs python libraries for a _single_ python version.  All our *.py scripts should use that version.

See INSTALL/README.md for details on install script.

# Dowload input from AWS S3 storage

Choose an organ whose samples you want to process:

```
aws s3 ls s3://endogene/mice_wgs/Aging/
aws s3 ls s3://endogene/mice_wgs/Aging/Old-liver/
mkdir Old-liver
aws s3 cp s3://endogene/mice_wgs/Aging/Old-liver/old_liver_R1_001.fastq.gz Old-liver/
```

Now repeat for `_R2_` and for both Old and Young samples — but if you're short on disk space, you'll want to next steps for each one so you can delete inputs...

## streaming

The command supports `-` as either source (meaning stdin) or destination (meaning stdout):
```
aws s3 cp s3://endogene/mice_wgs/Aging/Old-liver/old_liver_R1_001.fastq.gz - | gunzip --stdout | head --lines=20
```

# breaking into chunks

for example in p53-liver
```sh
./split_recompress.py p53-liver_R1_001.fastq.gz p53-liver_R1_001
./split_recompress.py p52-liver_R2_001.fastq.gz p53-liver_R2_001
...
```
This cuts input into chunks of 100 000 000 (1e8) lines - which are exactly 25_000_000 records - writing files named:
```
p53-liver_R1_001.part0e8.fastq.gz
p53-liver_R1_001.part1e8.fastq.gz
...
p53-liver_R1_001.part27e8.fastq.gz
...
```
(last chunk will be smaller.)

This allows following steps to be run from the middle and/or parallelized.

## Tests
```sh
python3.6 -m doctest --option=ELLIPSIS split_recompress.py
```

# r1r2merge.py

To support streaming, this script no longer takes an output file name;
it now always writes to stdout, *without compression*.

To merge the whole input:
```
./r1r2merge.py Young-liver/wt-liver_R1_001.fastq.gz Young-liver/wt-liver_R2_001.fastq.gz | gzip --stdout -2 > Young-liver/wt-liver_merged.fastq.gz
```

But we can run it faster by parallelizing over chunks produced above:
You should replace 2nd argument of `seq` with number of chunks you have.
`--jobs=-2` means number of CPU cores you have minus 2.
```
seq 1 10 | parallel --jobs=-2 --eta --joblog=Young-liver/merge.joblog './r1r2merge.py Young-liver/wt-liver_R1_001.part{}e8.fastq.gz Young-liver/wt-liver_R2_001.part{}e8.fastq.gz | gzip --stdout -2 > Young-liver/wt-liver_merged.part{}e8.fastq.gz'
```

## Tests

```sh
./r1r2merge.py merge-test-R1.fastq merge-test-R2.fastq > merge-test-merged.fastq
diff --report-identical-files merge-test-merged.fastq merge-test-expected.fastq
```

## Process

- Currently, run_part_1.py is the high level script. This is messy and should be refactored in the future
- To generate potential sines run mode = 1.
- To generate barcodes run mode = 3
- cat $(ls -v old_liver_merged*/old_liver_merged.part*e8_sineBarcode.fastq.gz) > old_liver_merged_sineBarcode.fastq.gz
