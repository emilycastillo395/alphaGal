# FAAH2a Off Target Analysis
Started 13May2026

Emily I. Thornton

## Set Up
### 1. Make Working Directory
`mkdir alphaGal_Workspace` This is what I named my working directory for this project. It cn be anything you want! As long as your file name does not have any spaces!  

### 2. Secure Copy File to working directory  
Our data is typically from Quintara, which can be downloaded by going to their website

Copy files to your working directory.

Go into your local terminal by opening another terminal screen and direct yourself to the folder or file you want to port over. Use the `scp` command to port your files over.  

Example:
`scp Clean_Data.zip emilycastillo395@grace.hprc.tamu.edu:/scratch/user/emilycastillo395/alphaGal_Workspace`

## Minimap2

### 1. Set Up
Create the following directories in a working directory space and pull in your respective fastq and fasta files.
```
mkdir fastq_files
mkdir reference_fastas
```
Make a .sh file called `minimap2.sh` by copying the command below:
```
touch minimap2.sh
```
Paste in the following bash command for hprc. Edit your `REFERENCE FASTA FILE` and your `DIRECTORIES` with the appropriate pathway
```
#!/bin/bash
#SBATCH --job-name=minimap2
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=16G
#SBATCH --time=04:00:00

####### LOAD DEPENDENCIES
module purge
module load GCCcore/13.2.0 minimap2/2.29

####### REFERENCE FASTA FILE
REFERENCE="/scratch/user/emilycastillo395/FAAH2a_workspace/reference_fastas/DrerioFAAH2a_Reference.fa"
INDEX="${REFERENCE%.*}.mmi"

####### DIRECTORIES
FASTQ_FILES="/scratch/user/emilycastillo395/FAAH2a_workspace/fastq_files/"
OUT_DIR="/scratch/user/emilycastillo395/FAAH2a_workspace/output_sam/"
LOG_DIR="/scratch/user/emilycastillo395/FAAH2a_workspace/log_files/"

mkdir -p "$OUT_DIR" "$LOG_DIR"

####### BUILD INDEX
minimap2 -d "$INDEX" "$REFERENCE" > "$LOG_DIR/index.log" 2>&1

####### RUN MINIMAP2
for FILE in "$FASTQ_FILES"/*.fastq.gz; do
  BASENAME=$(basename "$FILE" .fastq.gz)
  OUTPUT="$OUT_DIR/${BASENAME}.sam"
  LOGFILE="$LOG_DIR/${BASENAME}.log"

  minimap2 -ax map-ont "$INDEX" "$FILE" -o "$OUTPUT" > "$LOGFILE" 2>&1
done
```
### 2. Run Minimap2
Run the following command:
```
sbatch minimap2.sh
```
## Samtools
### 1. Set Up
Make a .sh file called `samtools.sh` by copy and pasting the following command into your working directory:
```
touch samtools.sh
```
Copy the following script into `samtools.sh`:

```
#!/bin/bash
#SBATCH --job-name=samtools
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=16G
#SBATCH --time=04:00:00

####### LOAD DEPENDENCIES
module purge
module load GCC/13.2.0 GCC/13.3.0 SAMtools/1.21

####### REFERENCE FASTA FILE
REFERENCE="/scratch/user/emilycastillo395/FAAH2a_workspace/reference_fastas/DrerioFAAH2a_Reference.fa"

####### DIRECTORIES
SAM_DIR="/scratch/user/emilycastillo395/FAAH2a_workspace/output_sam"
BAM_DIR="/scratch/user/emilycastillo395/FAAH2a_workspace/output_bam"
SORTED_BAM_DIR="/scratch/user/emilycastillo395/FAAH2a_workspace/sorted_bam"
LOG_DIR="/scratch/user/emilycastillo395/FAAH2a_workspace/log_files"

mkdir -p "$BAM_DIR" "$SORTED_BAM_DIR" "$LOG_DIR"

####### INDEX REFERENCE FASTA
samtools faidx "$REFERENCE" > "$LOG_DIR/reference_faidx.log" 2>&1

####### CONVERT SAM TO BAM
for FILE in $SAM_DIR/*.sam; do
  BASENAME=$(basename "$FILE" .sam)
  BAM_OUTPUT="$BAM_DIR/${BASENAME}.bam"
  LOGFILE="$LOG_DIR/${BASENAME}_samtools.log"

  samtools view -b "$FILE" > "$BAM_OUTPUT" 2> "$LOGFILE"
done

####### SORT BAM
for FILE in "$BAM_DIR"/*.bam; do
  BASENAME=$(basename "$FILE" .bam)
  SORTED_OUTPUT="$SORTED_BAM_DIR/${BASENAME}_sorted.bam"
  LOGFILE="$LOG_DIR/${BASENAME}_sort.log"

  samtools sort -o "$SORTED_OUTPUT" "$FILE" > "$LOGFILE" 2>&1
done

####### INDEX SORTED BAM

for FILE in "$SORTED_BAM_DIR"/*.bam; do
  BASENAME=$(basename "$FILE" .bam)
  LOGFILE="$LOG_DIR/${BASENAME}_index.log"ls

  samtools index "$FILE" > "$LOGFILE" 2>&1
done
```
### 2. Run Samtools
Run using the following
```
sbatch samtools.sh
```

## IGV Visialization
Open IGV at the website: https://igv.org/app/

### 1. Upload Reference sequences
Click on the `Genome` dropdown.
Select `Local File...`
Upload your `.fa` and `.fai` files at the same time.

### 2. Upload Sample bam and bai files
Click on the `Tracks` dropdown.
Select `Local File ...`
Upload your `.sorted.bam` and `sorted.bai` files together for each sample. One sample at a time.

