# AlphaGal Notebook
### Date: 2026-05-12

## Project context
Worked on paired-end alignment and downstream BAM/BAI generation for GGTA and PMEL samples. The HPRC working directory was `/scratch/user/emilycastillo395/alphaGal_Workspace/Clean_Data/`.

### 1. Cutadapt and paired-end reads
Confirmed that `cutadapt` can process paired-end `R1` and `R2` reads together when both files are provided in the same command. Output files were written with new names so the original FASTQ files were not overwritten. The Illumina universal adapter sequence was used with paired-end flags: `-a AGATCGGAAGAGC` for `R1` and `-A AGATCGGAAGAGC` for `R2`. Trimmed reads were written to `trimmed_fastq/`.

### 2. BWA alignment script setup
Updated the BWA loops to run from the parent directory `Clean_Data` while reading files from `trimmed_fastq/`. Confirmed that BWA does not automatically detect mate pairs from filenames, so both `R1` and `R2` must be passed explicitly. The filename pairing pattern used `*.R1.trimmed.fastq.gz` and derived the mate by replacing `.R1.trimmed.fastq.gz` with `.R2.trimmed.fastq.gz`. SAM outputs were written into `sam_files/`.

### 3. Logging
Added `exec > bwa_align.log 2>&1` to send script standard output and standard error to `bwa_align.log`. Clarified that SAM output still goes to `sam_files/*.sam`, while BWA progress messages and errors go to `bwa_align.log`.

### 4. Initial zero-byte SAM problem
Zero-byte SAM files indicated that `bwa mem` was not producing alignment output. The main possible causes considered were an incorrect file glob, wrong `R1/R2` pairing, wrong reference path, or a BWA failure with the error written to standard error. Found and corrected inconsistent reference paths. Also noted that the relative path `scratch/...` was missing the leading slash when written as an absolute path. Standardized on running from `Clean_Data` with relative reference paths: `reference_sequences/GGTA1.fa` and `reference_sequences/PMEL.fa`.

### 5. Samtools sorting and indexing
Updated the samtools script to run from `Clean_Data` while reading SAM files from `sam_files/`. Created `bam_files/` and directed BAM and BAI outputs there. Fixed a path bug where `base=${file%.sam}` preserved `sam_files/` in the basename and caused invalid output paths such as `bam_files/sam_files/GGTA_1-control.bam`. Corrected this by using `base=$(basename "$file" .sam)`.

### 6. BAI file appearance
Confirmed that `.bai` files are binary, so unreadable characters are normal when opened as text. This did not indicate corruption by itself.

### 7. IGV troubleshooting
IGV loaded the reference but did not display alignments for sample tracks. Considered the index naming issue, since IGV often expects `sample.sorted.bam.bai` and may not automatically detect `sample.sorted.bai`. A copied index with the `.bam.bai` name can help automatic detection. However, this was not the root cause once alignment statistics were checked.

### 8. Alignment statistics
Ran `samtools idxstats` on the sorted BAM files. The output consistently showed zero mapped reads to the GGTA1 or PMEL references and all reads listed as unmapped under `*`. The conclusion was that the BAM files were valid, but the reads did not align to the provided references.

### 9. Read/reference sequence comparison
Examined a representative read from `trimmed_fastq/GGTA_1-control.R1.trimmed.fastq.gz`. The read began with `GATGAGGCTAGTTGGTTGAACTACCGATGTGAATTCAGATATTCATTCAGATTAAATTCAGAAATGTCAGAAACATACAC`. Examined the start of `reference_sequences/GGTA1.fa` and found that the reference began with a very different GC-rich sequence that did not resemble the read. Searched the read sequence in `GGTA1.fa` and found no match. The conclusion was that the read sequence is not present in `GGTA1.fa`.

### 10. Case sensitivity question
Confirmed that case of nucleotide letters is not the issue. BWA treats uppercase and lowercase bases the same for alignment. Case can matter for file paths and reference names, but not for the DNA sequence content itself.

### 11. Final troubleshooting conclusion
The alignment commands and downstream BAM generation were functioning. The main problem was not IGV, samtools, or letter case. The main problem was a reference-target mismatch: the reads are real and high quality, the BAM files are valid, all reads are unmapped, and the read sequence is not present in `GGTA1.fa`. The most likely next step is to identify and align against the exact intended amplicon or target reference sequence for these reads.

## Key commands used during troubleshooting
### Check one read:
`zcat trimmed_fastq/GGTA_1-control.R1.trimmed.fastq.gz | sed -n '2p' | cut -c1-80`

### Check reference start:
`head reference_sequences/GGTA1.fa`

### Check BAM mapping stats:
`samtools idxstats bam_files/GGTA_1-control.sorted.bam`

### Check all BAM mapping stats:
`for f in bam_files/*.sorted.bam; do echo "$f"; samtools idxstats "$f"; done`

### Search for a read sequence in the reference:
`grep -i "READ_SEQUENCE" reference_sequences/GGTA1.fa`

### Recommended next step
Obtain or construct the exact reference or amplicon sequence expected for the GGTA and PMEL reads, then rerun BWA against that reference.

