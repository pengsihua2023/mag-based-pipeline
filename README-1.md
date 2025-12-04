# Viral Metagenome Analysis Pipeline

Complete pipeline for viral metagenome assembly and classification using nf-core/mag and direct containerized tools.

## 📋 Overview

This repository contains two independent workflows for analyzing viral metagenomes:

1. **Short Reads Analysis** - Using nf-core/mag v2.5.4 (Illumina paired-end data)
2. **Long Reads Analysis** - Using direct Flye + Kraken2 containers (Nanopore/PacBio data)

## 🚀 Quick Start

### Prerequisites

- SLURM cluster access
- Conda/Miniforge3 with nextflow_env activated
- Apptainer/Singularity installed
- Kraken2 viral database

### Short Reads Analysis

```bash
# Submit short reads analysis
sbatch run_short_reads.sh
```

**Output:** `results_short/`

### Long Reads Analysis

```bash
# Submit long reads analysis
sbatch run_long_reads.sh
```

**Output:** `results_long/`

## 📁 File Structure

```
.
├── run_short_reads.sh                    # SLURM script for short reads (nf-core/mag)
├── run_long_reads.sh                     # SLURM script for long reads (direct containers)
├── nfcore_mag_v3_no_validation.config    # Configuration for nf-core/mag
├── samplesheet_mag_short_reads.csv       # Short reads sample sheet template
├── samplesheet_mag_long_reads.csv        # Long reads sample sheet template
├── verify_input_files.sh                 # Input file verification script
├── DATABASE_SETUP.md                     # Kraken2 database setup guide
└── README.md                             # This file
```

## 🧬 Pipeline Details

### Short Reads Workflow (nf-core/mag v2.5.4)

**Tools:**
- fastp: Quality control and adapter trimming
- MEGAHIT: Fast metagenomic assembly
- metaSPAdes: High-quality metagenomic assembly
- Kraken2: Taxonomic classification

**Steps:**
1. Quality control (fastp)
2. PhiX removal (Bowtie2)
3. Parallel assembly (MEGAHIT + metaSPAdes)
4. Taxonomic classification (Kraken2 on reads and contigs)
5. Report generation (MultiQC)

**Runtime:** ~4-8 hours

**Resources:**
- CPUs: 32
- Memory: 256 GB
- Time: 72 hours (max)

### Long Reads Workflow (Direct Containers)

**Tools:**
- Flye: Metagenomic assembly
- Kraken2: Taxonomic classification

**Steps:**
1. Flye metagenomic assembly
2. Kraken2 classification on assembled contigs
3. Kraken2 classification on raw reads

**Runtime:** ~2-4 hours

**Resources:**
- CPUs: 32
- Memory: 256 GB
- Time: 24 hours (max)

## 📊 Output Structure

### Short Reads Output

```
results_short/
├── QC_shortreads/
│   ├── *_fastp.html                      # Quality control reports
│   └── *_fastp.json
├── Assembly/
│   ├── *_MEGAHIT.fa.gz                   # MEGAHIT assemblies
│   └── *_SPAdes.fa.gz                    # SPAdes assemblies
├── Taxonomy/
│   ├── *_reads.kraken2.report.txt        # Read-level classification
│   ├── *_MEGAHIT.kraken2.report.txt      # MEGAHIT contig classification
│   └── *_SPAdes.kraken2.report.txt       # SPAdes contig classification
├── execution_report.html                  # Workflow execution report
└── execution_timeline.html                # Timeline visualization
```

### Long Reads Output

```
results_long/
├── flye_assembly/
│   ├── assembly.fasta                     # Assembled contigs
│   ├── assembly_info.txt                  # Assembly statistics
│   └── assembly_graph.gfa                 # Assembly graph
├── llnl_66d1047e_contigs_kraken2_report.txt    # Contig classification
└── llnl_66d1047e_reads_kraken2_report.txt      # Read classification
```

## 🗄️ Database Setup

### Kraken2 Viral Database

The pipeline uses a pre-built Kraken2 viral database located at:
```
/scratch/sp96859/Meta-genome-data-analysis/Apptainer/databases/kraken2_Viral_ref
```

For detailed database setup instructions, see [`DATABASE_SETUP.md`](DATABASE_SETUP.md).

**Quick setup (if needed):**

```bash
# Standard Viral database (~500 MB)
DB_DIR="/scratch/sp96859/Meta-genome-data-analysis/Apptainer/databases/kraken2_viral"
mkdir -p $DB_DIR
kraken2-build --download-library viral --db $DB_DIR --threads 32
kraken2-build --download-taxonomy --db $DB_DIR
kraken2-build --build --db $DB_DIR --threads 32
kraken2-build --clean --db $DB_DIR
```

## 📝 Sample Data

### Current Samples

**Short reads:**
- Sample: `llnl_66ce4dde`
- R1: `/scratch/sp96859/Meta-genome-data-analysis/Apptainer/data/short_reads/llnl_66ce4dde_R1.fastq.gz` (639M)
- R2: `/scratch/sp96859/Meta-genome-data-analysis/Apptainer/data/short_reads/llnl_66ce4dde_R2.fastq.gz` (653M)

**Long reads:**
- Sample: `llnl_66d1047e`
- Reads: `/scratch/sp96859/Meta-genome-data-analysis/Apptainer/data/long_reads/llnl_66d1047e.fastq.gz` (252M)

### Adding New Samples

**For short reads:**

Edit `samplesheet_short.csv` in `run_short_reads.sh`:
```csv
sample,group,short_reads_1,short_reads_2,long_reads
sample1,viral_group1,/path/to/R1.fastq.gz,/path/to/R2.fastq.gz,
sample2,viral_group1,/path/to/R1.fastq.gz,/path/to/R2.fastq.gz,
```

**For long reads:**

Update variables in `run_long_reads.sh`:
```bash
LONG_READS="/path/to/your/long_reads.fastq.gz"
SAMPLE="your_sample_name"
```

## 🔧 Troubleshooting

### Issue: /lscratch Mount Error

**Error:**
```
FATAL: container creation failed: mount /lscratch->/lscratch error
```

**Solution:**
Already fixed in `nfcore_mag_v3_no_validation.config` with:
```groovy
apptainer {
    autoMounts = false
    runOptions = '--no-mount /lscratch --bind /scratch:/scratch --bind /home:/home'
}
```

### Issue: Disk Quota Exceeded

**Error:**
```
disk quota exceeded
```

**Solution:**
Set Apptainer cache to `/scratch`:
```bash
export APPTAINER_CACHEDIR="/scratch/sp96859/Meta-genome-data-analysis/Apptainer/cache"
export APPTAINER_TMPDIR="/scratch/sp96859/Meta-genome-data-analysis/Apptainer/tmp"
```

Already configured in `run_long_reads.sh`.

### Issue: Container Pull Failed

**Solution:**
```bash
# Clean Apptainer cache
rm -rf /home/$USER/.apptainer/cache/oci-tmp/*

# Or set cache to /scratch (already in scripts)
export APPTAINER_CACHEDIR="/scratch/path/to/cache"
```

## 📈 Results Analysis

### View Kraken2 Reports

**Short reads:**
```bash
# View top viral classifications
awk '$4=="D" || $4=="F" {printf "%6.2f%% - %s\n", $1, $6}' \
    results_short/Taxonomy/*_reads.kraken2.report.txt | head -20

# Compare MEGAHIT vs SPAdes
grep "Viruses" results_short/Taxonomy/*_MEGAHIT.kraken2.report.txt
grep "Viruses" results_short/Taxonomy/*_SPAdes.kraken2.report.txt
```

**Long reads:**
```bash
# View top viral classifications from contigs
awk '$4=="D" || $4=="F" {printf "%6.2f%% - %s\n", $1, $6}' \
    results_long/llnl_66d1047e_contigs_kraken2_report.txt | head -20

# View read-level classification
awk '$4=="D" || $4=="F" {printf "%6.2f%% - %s\n", $1, $6}' \
    results_long/llnl_66d1047e_reads_kraken2_report.txt | head -20
```

### View Assembly Statistics

**Short reads:**
```bash
# Check assembly quality
ls -lh results_short/Assembly/

# View execution report (open in browser)
firefox results_short/execution_report.html
```

**Long reads:**
```bash
# View Flye assembly info
cat results_long/flye_assembly/assembly_info.txt

# Check assembly stats
grep -c "^>" results_long/flye_assembly/assembly.fasta  # Contig count
```

## 🛠️ Advanced Usage

### Resume Failed Runs

Both workflows support resuming from the last successful step:

```bash
# Already enabled with -resume flag in both scripts
sbatch run_short_reads.sh  # Will auto-resume if previous run failed
```

### Adjust Resources

Edit SLURM header in the scripts:

```bash
#SBATCH --cpus-per-task=32    # Number of CPUs
#SBATCH --mem=256G            # Memory allocation
#SBATCH --time=72:00:00       # Maximum runtime
```

### Change Kraken2 Database

Edit the database path in scripts:

```bash
# For short reads (in run_short_reads.sh)
KRAKEN2_DB="/path/to/your/kraken2/db"

# For long reads (in run_long_reads.sh)
KRAKEN2_DB="/path/to/your/kraken2/db"
```

## 📚 Documentation

- **Database Setup:** [`DATABASE_SETUP.md`](DATABASE_SETUP.md)
- **nf-core/mag Documentation:** https://nf-co.re/mag/
- **Flye Documentation:** https://github.com/fenderglass/Flye
- **Kraken2 Manual:** https://github.com/DerrickWood/kraken2/wiki

## 💡 Best Practices

1. **Verify input files** before running:
   ```bash
   bash verify_input_files.sh
   ```

2. **Monitor progress:**
   ```bash
   # Check SLURM queue
   squeue -u $USER
   
   # View real-time output
   tail -f nfcore_mag_short_*.out
   tail -f long_reads_flye_*.out
   ```

3. **Save results:**
   ```bash
   # Copy important results
   cp -r results_short /path/to/backup/
   cp -r results_long /path/to/backup/
   ```

4. **Clean work directories** after successful completion:
   ```bash
   # Clean Nextflow work directory (after confirming results)
   rm -rf work_short/
   
   # Clean Flye intermediate files
   rm -rf results_long/flye_assembly/00-assembly/
   ```

## 🆘 Support

### Check Logs

**Short reads:**
- Output: `nfcore_mag_short_*.out`
- Error: `nfcore_mag_short_*.err`
- Nextflow: `.nextflow.log`

**Long reads:**
- Output: `long_reads_flye_*.out`
- Error: `long_reads_flye_*.err`

### Common Issues

| Issue | Solution |
|-------|----------|
| /lscratch mount error | Fixed in config file |
| Disk quota exceeded | Cache set to /scratch |
| Container not found | Check container version |
| Out of memory | Increase SLURM memory |

## 📊 Performance

### Short Reads (llnl_66ce4dde)
- **Input:** 1.3 GB (paired-end)
- **Runtime:** ~1h 14m
- **Processes:** 15
- **CPU hours:** 16.8

### Long Reads (llnl_66d1047e)
- **Input:** 252 MB
- **Runtime:** ~2-4 hours (estimated)
- **Assembly:** Flye metagenomic mode
- **CPU hours:** ~50-100

## 🎯 Citation

If you use this pipeline, please cite:

**nf-core/mag:**
```
Kieser, S., Brown, J., Zdobnov, E. M., Trajkovski, M. & McCue, L. A. 
ATLAS: a Snakemake workflow for assembly, annotation, and genomic binning of metagenome sequence data.
BMC Bioinformatics 21, 257 (2020).
https://doi.org/10.1186/s12859-020-03585-4
```

**Flye:**
```
Kolmogorov, M., Yuan, J., Lin, Y. et al.
Assembly of long, error-prone reads using repeat graphs.
Nat Biotechnol 37, 540–546 (2019).
https://doi.org/10.1038/s41587-019-0072-8
```

**Kraken2:**
```
Wood, D.E., Lu, J. & Langmead, B.
Improved metagenomic analysis with Kraken 2.
Genome Biol 20, 257 (2019).
https://doi.org/10.1186/s13059-019-1891-0
```

## 📝 Version Information

- **nf-core/mag:** v2.5.4
- **Nextflow:** v25.04.7
- **Flye:** v2.9.1 (containerized)
- **Kraken2:** v2.1.2 (containerized)
- **MEGAHIT:** v1.2.9 (containerized)
- **metaSPAdes:** v3.15.3 (containerized)

## 🔄 Workflow Comparison

| Feature | Short Reads | Long Reads |
|---------|-------------|------------|
| Tool | nf-core/mag | Direct containers |
| Assembly | MEGAHIT + SPAdes | Flye |
| QC | fastp + FastQC | Skipped (NumPy issue) |
| Classification | Kraken2 (reads + contigs) | Kraken2 (reads + contigs) |
| Binning | Skipped | Not applicable |
| Runtime | ~1-8 hours | ~2-4 hours |
| Automation | Full pipeline | Manual steps |

## 🎓 Understanding Results

### Kraken2 Report Format

```
 %reads  reads  direct  rank  tax_id  name
  45.23  15234    234    D     10239   Viruses
  12.45   4123   1234    P    439488     ssRNA viruses
   8.34   2765    876    F     11118       Flaviviridae
```

**Columns:**
- `%reads`: Percentage of reads classified to this taxon
- `reads`: Number of reads assigned (cumulative)
- `direct`: Number of reads directly assigned
- `rank`: Taxonomic rank (D=Domain, P=Phylum, F=Family, G=Genus, S=Species)
- `tax_id`: NCBI taxonomy ID
- `name`: Scientific name

### Assembly Quality Metrics

**Short reads (MEGAHIT/SPAdes):**
- Contig count
- N50 length
- Total assembly size
- Longest contig

**Long reads (Flye):**
- See `results_long/flye_assembly/assembly_info.txt`
- Check assembly graph: `assembly_graph.gfa`

## 🔬 Analyzing Your Results

### Extract Viral Sequences

```bash
# From short reads (example: extract all Flaviviridae)
awk '$6~/Flaviviridae/ {print $2}' \
    results_short/Taxonomy/*_classification.txt > flaviviridae_read_ids.txt

# Extract reads by ID (requires seqtk)
seqtk subseq input.fastq flaviviridae_read_ids.txt > flaviviridae_reads.fastq
```

### Compare Assemblers

```bash
# Compare viral detection between MEGAHIT and SPAdes
echo "=== MEGAHIT ==="
grep "Viruses" results_short/Taxonomy/*_MEGAHIT.kraken2.report.txt

echo "=== SPAdes ==="
grep "Viruses" results_short/Taxonomy/*_SPAdes.kraken2.report.txt
```

### Generate Summary Statistics

```bash
# Total viral reads (short reads)
awk '$6~/Viruses/ {sum+=$2} END {print "Total viral reads:", sum}' \
    results_short/Taxonomy/*_reads.kraken2.report.txt

# Total viral contigs (long reads)
awk '$6~/Viruses/ {sum+=$2} END {print "Total viral contigs:", sum}' \
    results_long/llnl_66d1047e_contigs_kraken2_report.txt
```

## 🛡️ Important Notes

### Container Cache

- Short reads: Uses `/scratch/.../Apptainer/singularity` (set via NXF_SINGULARITY_CACHEDIR)
- Long reads: Uses `/scratch/.../Apptainer/cache` (set via APPTAINER_CACHEDIR)

This avoids home directory disk quota issues.

### Known Limitations

1. **nf-core/mag v2.5.4:**
   - Cannot process long-reads-only samples
   - Strict validation requires short_reads_1 to be non-empty

2. **Long reads direct pipeline:**
   - NanoPlot QC skipped due to NumPy compatibility
   - Manual pipeline (not automated by nf-core/mag)

### Why Two Separate Pipelines?

- nf-core/mag v2.5.4 is stable but doesn't support long-reads-only
- nf-core/mag v3.x supports mixed reads but has strict parameter validation
- Direct container approach bypasses all validation issues

## 📧 Contact

For issues or questions, please check:
- `.nextflow.log` for detailed error messages
- SLURM error logs: `*_%j.err`
- Execution reports: `results_*/execution_report.html`

## 📄 License

This pipeline uses open-source tools. Please cite appropriately.

---

**Last updated:** December 3, 2025


