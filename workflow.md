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

### Clean Up FASTA Files
This is what biopython has on their website. It's a script called `sequence_cleaner`. "the big idea is to remove duplicate sequences, remove too short sequences (the user defines the minimum length) and remove sequences which have too many unknown nucleotides (N) (the user defines the % of N it allows ) and in the end the user can choose if he/she wants to have a file as output or print the result."

```
import sys
from Bio import SeqIO


def sequence_cleaner(fasta_file, min_length=0, por_n=100):
    # Create our hash table to add the sequences
    sequences = {}

    # Using the Biopython fasta parse we can read our fasta input
    for seq_record in SeqIO.parse(fasta_file, "fasta"):
        # Take the current sequence
        sequence = str(seq_record.seq).upper()
        # Check if the current sequence is according to the user parameters
        if (
            len(sequence) >= min_length
            and (float(sequence.count("N")) / float(len(sequence))) * 100 <= por_n
        ):
            # If the sequence passed in the test "is it clean?" and it isn't in the
            # hash table, the sequence and its id are going to be in the hash
            if sequence not in sequences:
                sequences[sequence] = seq_record.id
            # If it is already in the hash table, we're just gonna concatenate the ID
            # of the current sequence to another one that is already in the hash table
            else:
                sequences[sequence] += "_" + seq_record.id

    # Write the clean sequences

    # Create a file in the same directory where you ran this script
    with open("clear_" + fasta_file, "w+") as output_file:
        # Just read the hash table and write on the file as a fasta format
        for sequence in sequences:
            output_file.write(">" + sequences[sequence] + "\n" + sequence + "\n")

    print("CLEAN!!!\nPlease check clear_" + fasta_file)


userParameters = sys.argv[1:]

try:
    if len(userParameters) == 1:
        sequence_cleaner(userParameters[0])
    elif len(userParameters) == 2:
        sequence_cleaner(userParameters[0], float(userParameters[1]))
    elif len(userParameters) == 3:
        sequence_cleaner(
            userParameters[0], float(userParameters[1]), float(userParameters[2])
        )
    else:
        print("There is a problem!")
except:
    print("There is a problem!")
```

### Generate your Consensus Sequence for each sequence within the FASTA file
To send this to the hprc, I created a sbatch job called `mafft_consensus`. Here is the contents of the sbatch command. This allows each file to run simultaneously. When submistted there are 16 jobs submitted to the hprc. You can visualize the status of these jobs at `squeue --me`.
```
#!/bin/bash
#SBATCH --job-name=mafft_consensus
#SBATCH --output=logs/%x_%A_%a.out
#SBATCH --error=logs/%x_%A_%a.err
#SBATCH --time=12:00:00
#SBATCH --cpus-per-task=4
#SBATCH --ntasks=1
#SBATCH --mem=8G
#SBATCH --array=0-16

echo "Starting job $SLURM_ARRAY_TASK_ID"

module load Anaconda3/2024.02-1

# initialize conda for non-interactive shell
source $(conda info --base)/etc/profile.d/conda.sh

conda activate myenv

# Get file list (sorted for consistency)
mapfile -t files < <(ls *.fasta.gz | sort)
f=${files[$SLURM_ARRAY_TASK_ID]}

echo "Processing: $f"

base="${f/.fasta.gz/}"
tmp_aln="${base}_aligned.fasta"
out="${base}_consensus.fasta"

# Step 1: Align with MAFFT
gunzip -c "$f" | mafft --auto --thread 4 - > "$tmp_aln"

# Step 2: Generate consensus with EMBOSS
cons -sequence "$tmp_aln" -outseq "$out"

echo "DONE: $out"
```
