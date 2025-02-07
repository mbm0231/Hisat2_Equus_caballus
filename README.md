# Hisat2_Equus_caballus
1. Filter out gap-only sequences
2. Create a cleaned FASTA file
3. Use the cleaned file for HISAT2 indexing
```
#!/bin/bash

#SBATCH --time=24:00:00
#SBATCH --job-name=Hisat2
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=8
#SBATCH --partition=jumbo
#SBATCH --mem=256GB
#SBATCH --mail-type=ALL
#SBATCH --mail-user=mbmo231@uky.edu,farman@uky.edu
#SBATCH -A cea_farman_s24cs485g
#SBATCH -o /project/farman_s24cs485g/mbmo231/alignments/%x_%J.out

# Error handling
set -e
set -u
set -o pipefail

# Configuration
readonly THREADS=8
readonly PROJECT_DIR="/project/farman_s24cs485g/mbmo231"
readonly REF_DIR="${PROJECT_DIR}/Horse_ref"
readonly ALIGN_DIR="${PROJECT_DIR}/alignments"
readonly LOG_DIR="${PROJECT_DIR}/logs"
readonly RAW_READS_DIR="${PROJECT_DIR}/RNAseqRota"
readonly Hisat2image="/share/singularity/images/ccs/conda/amd-conda2-centos8.sinf"
readonly SAMTOOLS='singularity run --app samtools113 /share/singularity/images/ccs/ngstools/samtools-1.13+matplotlib-bcftoools-1.13.sinf samtools'
readonly indexname='GCF_002863925.1_EquCab3.0_genomic'
readonly fasta="${indexname}.fna.gz"
readonly cleaned_fasta="${indexname}.cleaned.fna"
readonly indexdir="${REF_DIR}/${indexname}_index"

# Input files
readonly R1="${RAW_READS_DIR}/1_S27_R1_001.fastq.gz"
readonly R2="${RAW_READS_DIR}/1_S27_R2_001.fastq.gz"

# Create directories
mkdir -p "$ALIGN_DIR" "$LOG_DIR" "$RAW_READS_DIR" "$indexdir"

# Logging setup
exec 1> >(tee "${LOG_DIR}/$(date +%Y%m%d_%H%M%S)_hisat2.log") 2>&1

# Clean FASTA file - remove sequences with only gaps
echo "Cleaning FASTA file..."
zcat "${REF_DIR}/${fasta}" | awk \'!/^>/ {gsub(/[^ATCGN]/,"N")} {print}\' | awk \'BEGIN{RS=">";FS="\\
"}NR>1{seq="";for(i=2;i<=NF;i++){seq=seq $i} if(seq!~/^N*$/){print ">"$1"\\
"seq}}\' > "${REF_DIR}/${cleaned_fasta}"

# Input validation
if [ ! -f "${REF_DIR}/${cleaned_fasta}" ]; then
    echo "Error: Failed to create cleaned FASTA file"
    exit 1
fi

if [ ! -f "${R1}" ] || [ ! -f "${R2}" ]; then
    echo "Error: Raw read files not found"
    echo "Expected files:"
    echo "  - ${R1}"
    echo "  - ${R2}"
    exit 1
fi

# Print runtime information
echo "Starting HISAT2 index building at $(date)"
echo "Using cleaned FASTA file: ${REF_DIR}/${cleaned_fasta}"
echo "Output index directory: ${indexdir}"

# Build HISAT2 index using cleaned FASTA
singularity run --app hisat2221 "$Hisat2image" \
    hisat2-build \
    --large-index \
    -p "$THREADS" \
    "${REF_DIR}/${cleaned_fasta}" \
    "${indexdir}/${indexname}"

echo "HISAT2 index building completed at $(date)"

# Perform alignment
echo "Starting HISAT2 alignment at $(date)"
singularity run --app hisat2221 "$Hisat2image" hisat2 \
    -p "$THREADS" \
    -x "$indexdir/$indexname" \
    -1 "$R1" \
    -2 "$R2" \
    --summary-file "$ALIGN_DIR/align_summary.txt" \
    --dta-cufflink | \
    $SAMTOOLS sort -@ "$THREADS" -O bam -o "$ALIGN_DIR/accepted_hits.bam"

# Index the BAM file
$SAMTOOLS index "$ALIGN_DIR/accepted_hits.bam"

echo "HISAT2 alignment and BAM indexing completed at $(date)"
'''
# Save the script to a file
script_filename = "hisat2_cleaned_reference.sh"
with open(script_filename, "w") as script_file:
    script_file.write(script_content)
print(f"Updated HISAT2 script written to {script_filename}")
```
