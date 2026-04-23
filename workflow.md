# AlphaGal Sequence Alignment
## SetUp
### Log In to TAMU HPRC:
`ssh NedID@grace.hprc.tamu.edu`  

Enter password when prompted. This will also ask you for a duo push. Make sure you have your duo open as it sometimes does not send a push notification.  
  
When in the hprc, you will see your computer prompt look something like:  `[NetID@grace4 ~]`  
Do not work in this space!!!! Instead, go to your `/scratch/ ` directory found by typing `$SCRATCH`

### Make a directory to work in
`mkdir alphaGal_Workspace` This is what I named my working directory for this project. It cn be anything you want! As long as your file name does not have any spaces!  

### Secure Copy File to working directory  
To do this, go into your local terminal by opening another terminal screen and direct yourself to the folder or file you want to port over. Use the `scp` command to port your files over.  
`scp Clean_Data.zip emilycastillo395@grace.hprc.tamu.edu:/scratch/user/emilycastillo395/alphaGal_Workspace`
### Create Your Conda Enviornment  
To check which Anaconda versions are available use `module avail Anaconda3`.  I chose the most updated Anaconda which is Anaconda3/2024.02-1
#### Output:
```
------------------------------------------------ /sw/eb/mods/all/Core -------------------------------------------------
   Anaconda3/2021.05    Anaconda3/2021.11    Anaconda3/2023.09-0    Anaconda3/2024.02-1 (D)
```
Load in your Anaconda, configure and create your environment. If this does not have enough explanation, you can also search how to do this in the hprc.tamu.edu page.
```
module load Anaconda3/2024.02-1
conda config --set auto_activate_base False
conda create --name myenv
```
Here my environment name is `myenv`. If I want to reopen my environment, I can use `conda activate myenv`.  
Now, I need to open my environment:  
```
source activate myenv
```
### Install Conda Packages  
So far I am only using `seqtk`. As I go, I will add more.  
```
conda install -c bioconda seqtk mafft
```
## Check Quality of fastq files and Create and Organize FASTA Files
### Merge FASTQ R1 and R2 Files (Seqtk)
Note: Your R1 and R2 files are forward and reverse seqeunces from the sequencing machine. We need to merge these to make one long DNA sequence per read. To do this, we will use `seqtk`
```
for r1 in *.R1.fastq.gz; do r2="${r1/.R1.fastq.gz/.R2.fastq.gz}"; out="${r1/.R1.fastq.gz/_merged.fastq.gz}"; seqtk mergepe "$r1" "$r2" | gzip > "$out"; done &
```

### Run FastQC to quality check sequences
```
module load FastQC
mkdir fastqc_reports
fastqc -o fastqc_reports *_merged.fastq.gz
```
### Clean Up using Cutadapt
```
module load GCCcore/13.2.0 cutadapt/5.0
```
To clean up, I opted for discarding any sequence below a phred score of 20 (-q 20), an error rate above 10% (-e 0.1), and a minimum sequence length of 100bp (-m 100).
```
for f in *merged.fastq.gz; do base=${f%.fastq.gz}; cutadapt -a AGATCGGAAGAGC -e 0.1 -q 20 -m 100 -o trimmed_fastq/${base}.trimmed.fastq.gz "$f"; done
```
### Run FastQC to quality check sequences after cleanup
```
mkdir fastqc_reportsClean
fastqc -o fastqc_reportsClean *.trimmed.fastq.gz
```
### Convert FASTQ files to FASTA files
```
{ echo "Running..."; for f in ./trimmed_fastq/*_merged.fastq.gz; do echo "Processing $f"; out="${f/_merged.fastq.gz/.fasta}"; seqtk seq -a "$f" | gzip > "$out.gz"; done; echo "DONE!" } > fasta_files &

{ echo "Running..."; for f in ./trimmed_fastq/*.merged.trimmed.fastq.gz; do echo "Processing $f"; seqtk seq -a "$f"; done; echo "DONE!"; } > fasta_files &
```
## BWA or bowtie2 Alignment



```
