# Beenome selection analysis pipeline

This pipeline includes the [HyPhy](https://www.hyphy.org/) programs BUSTED and aBSREL.

Note: Run the pipeline on Atlas. The pipeline encounters errors on Ceres.

## 1. Orthogroup batch files
The ~8500 orthogroup files are split into smaller batches for processing. 

On Atlas the batch files are located at `/project/beenome100_collab/hyphy_orthogroups` 

For folks not working on Atlas, the files can be downloaded (see email for link).

To keep track of who is running what, record which batch you are processing in the Google Doc (see email for link).

## 2. Run Pipemake
Pipemake creates the Snakemake workflow for running BUSTED and aBSREL.

On Atlas do the following on the login node (Pipemake is fast and not resource intensive) because Apptainer won't load on an interactive node.

### 2.1 Load the Conda environment

On Atlas you should be able to access the Beenome shared conda environment directory:
```
module load miniconda3
source activate /project/beenome100_collab/conda_envs/hyphy_beenome_env
```

Otherwise, create the conda environment from the environment YAML file.
```
conda env create --file hyphy_beenome_env.yml
conda activate hyphy_beenome_env
```

The conda environment just has Pipemake, Snakemake, and the Snakemake SLURM executor. To only install Pipemake:
```
conda install -c bioconda kocherlab::pipemake=1.5.0
```

### 2.2 Load Apptainer
On Atlas `module load apptainer`

### 2.3 Run pipemake
```
pipemake msf-codon-selection --msf-wildcard /project/beenome100_collab/hyphy_orthogroups/random#_split_##/{samples}_random0.cds.fa \
    --busted-labels Bees \
    --outgroup-file /project/beenome100_collab/conda_envs/hyphy_files/outgroup_species.txt \
    --species-tree /project/beenome100_collab/conda_envs/hyphy_files/labeled_species_tree.tre \
    --singularity-dir /project/beenome100_collab/conda_envs/hyphy_singularity \
    --scale-threads 6 \
    --scale-mem 6 \
    --workflow-dir Selection_random#_split_##
```

For each orthogroup batch you will need to update: 
- `--msf-wildcard`
    - Update the path to the batch directory you are processing
    - Based on the batch file, update the number in `{samples}_random0.cds.fa` (random0, random1, or random2)
- `--workflow-dir`
    - Update with the name of the batch you are processing

## 2.4 Update the job script
Pipemake will print some text, including:
```
msf-codon-selection version 1.0 has been configured, please use the following command within the <workflow-dir> directory:
snakemake --use-singularity --singularity-args '--bind /path/to/orthogroup/sequence/files'
```

Update `'--bind /path/to/orthogroup/sequence/files'` in the slurm script.
<details>
<summary>job script</summary>

```
#!/bin/bash
#SBATCH --partition=atlas
#SBATCH --time=14-00:00:00
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=2
#SBATCH --mem=4G
#SBATCH --account=<your account>

module load apptainer
module load miniconda3
source activate /project/beenome100_collab/conda_envs/hyphy_beenome_env

snakemake --executor slurm --jobs 100 \
    --latency-wait 60 \
    --default-resources \
         slurm_account=<your account> \
         slurm_partition=atlas \
         runtime="14d" \
    --use-singularity \
    --singularity-args '--bind /path/to/orthogroup/sequence/files' \
    --keep-going  
```
</details>


## 3. Submit job
The `hyphy_beenome_env` conda environment includes Snakemake with the SLURM executor plugin.

On Atlas it takes about N days to run a batch. See the SLURM output file for progress updates.

### 3.1 Restarting a killed job
When restarting a killed pipeline job (1) Execute `snakemake --unlock` first so that the resubmitted job can write to the same directory as before. (2) Add `--rerun-incomplete` to the Snakemake command.

<details>
<summary>restart script</summary>

```
snakemake --unlock

snakemake --executor slurm --jobs 100 \
    --latency-wait 60 \
    --default-resources \
         slurm_account=<your account> \
         slurm_partition=atlas \
         runtime="14d" \
    --use-singularity \
    --singularity-args '--bind /path/to/orthogroup/sequence/files' \
    --keep-going \
    --rerun-incomplete
```
</details>

