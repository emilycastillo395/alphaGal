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
## Create and Organize FASTA Files
### Merge FASTQ R1 and R2 Files (Seqtk)
Note: Your R1 and R2 files are forward and reverse seqeunces from the sequencing machine. We need to merge these to make one long DNA sequence per read. To do this, we will use `seqtk`
```
for r1 in *.R1.fastq.gz; do r2="${r1/.R1.fastq.gz/.R2.fastq.gz}"; out="${r1/.R1.fastq.gz/_merged.fastq.gz}"; seqtk mergepe "$r1" "$r2" | gzip > "$out"; done &
```

### Convert FASTQ files to FASTA files
```
echo "Running..."; for f in *_merged.fastq.gz; do echo "Processing $f"; out="${f/_merged.fastq.gz/.fasta}"; seqtk seq -a "$f" | gzip > "$out.gz"; done; echo "DONE!" &
```
### Generate your Consensus Sequence for each sequence within the FASTA file
To send this to the hprc, I created a sbatch job called `mafft_consensus`. Here is the contents of the sbatch command. This allows each file to run simultaneously. When submistted there are 16 jobs submitted to the hprc. You can visualize the status of these jobs at `squeue --me`.
```
#!/bin/bash
#SBATCH --job-name=mafft_consensus
#SBATCH --output=mafft_consensus.out
#SBATCH --error=mafft_consensus.err
#SBATCH --time=04:00:00
#SBATCH --cpus-per-task=4
#SBATCH --ntasks=1
#SBATCH --array=0-16
#SBATCH --mem=8G

echo "Running..."

conda activate myenv

echo "Running..."; for f in *.fasta.gz; do echo "Processing $f"; gunzip -c "$f" | mafft --auto - | seqtk consensus > "${f/.fasta.gz/_consensus.fasta}"; done; echo "DONE!"
```
