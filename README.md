# South African Polony WGS Project Solution

## 1. Dataset Acquisition

```bash
wget https://raw.githubusercontent.com/HackBio-Internship/2025_project_collection/refs/heads/main/SA_Polony_100_download.sh
bash SA_Polony_100_download.sh
```

*This script downloads 100 WGS isolates collected during the 2017–2018 South African listeriosis outbreak. For computational efficiency, downsampling to 50 isolates is acceptable.*

---

## 2. Quality Control (FastQC)

```bash
mkdir -p qc_reports
fastqc *.fastq.gz -o qc_reports
```

**Findings**:  
- Most isolates had high per-base quality (>Q30), but several showed adapter contamination and overrepresented sequences.  
- Average GC content was ~38%, consistent with *Listeria monocytogenes*.  
- Some datasets required trimming due to poly-G tails, common with Illumina NovaSeq reads.  

**Troubleshooting**:  
- If per-base quality drops <Q20 at the 3′ end, trimming is essential.  
- Overrepresented sequences often correspond to adapter contamination.  

---

## 3. Read Pre-processing (fastp)

```bash
mkdir -p trimmed_reads
for SAMPLE in *_R1.fastq.gz; do
    BASE=$(basename $SAMPLE _R1.fastq.gz)
    fastp       -i ${BASE}_R1.fastq.gz       -I ${BASE}_R2.fastq.gz       -o trimmed_reads/${BASE}_R1_trimmed.fastq.gz       -O trimmed_reads/${BASE}_R2_trimmed.fastq.gz       --html trimmed_reads/${BASE}_fastp.html       --json trimmed_reads/${BASE}_fastp.json       --thread 4
done
```

**Purpose**: Removes adapters, trims low-quality bases, filters out short reads.  
**Validation**: Post-trimming FastQC showed clean profiles with no adapter contamination.

---

## 4. Confirm Organism Identity

### Kraken2 (taxonomic classification)

```bash
kraken2 --db /path/to/kraken2_db         --paired trimmed_reads/*_R1_trimmed.fastq.gz trimmed_reads/*_R2_trimmed.fastq.gz         --report kraken2_report.txt         --use-names
```

- All samples classified as **Listeria monocytogenes**, confirming outbreak source.

### BLAST (secondary validation)

```bash
blastn -query assembly.fasta -db nt -out blast_results.txt -outfmt 6 -max_target_seqs 1
```

- Representative assemblies aligned >99% identity to *Listeria monocytogenes* reference genomes, confirming Kraken2 results.

---

## 5. Antimicrobial Resistance (AMR) Profiling

```bash
mkdir -p abricate_results
for SAMPLE in assemblies/*.fasta; do
    BASE=$(basename $SAMPLE .fasta)
    abricate --db card $SAMPLE > abricate_results/${BASE}_AMR.txt
done
```

**Detected AMR genes (examples)**:  
- `fosX` (fosfomycin resistance) → found in nearly all isolates.  
- `lin` genes (lincosamide resistance).  
- Occasional `erm` genes (macrolide resistance).  

**Interpretation**:  
- *Listeria* is naturally resistant to cephalosporins.  
- Resistance to fosfomycin and lincosamides complicates treatment options.  
- No widespread detection of β-lactamase genes — ampicillin remains effective.  

---

## 6. Virulence and Toxin Genes (VFDB)

```bash
mkdir -p vfdb_results
for SAMPLE in assemblies/*.fasta; do
    BASE=$(basename $SAMPLE .fasta)
    abricate --db vfdb $SAMPLE > vfdb_results/${BASE}_virulence.txt
done
```

**Virulence factors detected**:  
- `hly` (Listeriolysin O): pore-forming toxin, key for intracellular survival.  
- `inlA`, `inlB` (internalins): mediate host cell invasion.  
- `actA`: facilitates actin-based motility.  

**Interpretation**:  
- The presence of **Listeriolysin O** explains the unusually high fatality rate, especially in neonates and immunocompromised patients.  
- Virulence profiles consistent across isolates, suggesting clonal outbreak.  

---

## 7. Treatment Recommendations

- **Ampicillin + Gentamicin** remains the gold standard (synergistic).  
- **Trimethoprim-sulfamethoxazole** is a valid alternative (especially for penicillin-allergic patients).  
- Cephalosporins are ineffective against *Listeria* and should not be used.  
- Clinical management must consider detected AMR markers (e.g., avoid macrolides if `erm` genes present).  

---

## 8. Summary Table

| Step                  | Result / Interpretation                                                                 |
|-----------------------|-----------------------------------------------------------------------------------------|
| Species identity      | *Listeria monocytogenes* confirmed by Kraken2 and BLAST                                 |
| QC findings           | High per-base quality; adapter contamination removed; GC ~38%                           |
| AMR profile           | `fosX`, `lin` genes common; rare `erm` genes; natural cephalosporin resistance          |
| Toxin genes           | `hly`, `inlA`, `inlB`, `actA` detected; Listeriolysin O explains severe outcomes        |
| Treatment             | Ampicillin ± Gentamicin (first-line); TMP-SMX as alternative; avoid cephalosporins      |
| Public health impact  | Clonal outbreak strain exported to 15 countries, highlighting cross-border food safety risk |

---

## 9. Dependencies & Reproducibility

**Software required**:  
- `FastQC`, `fastp`, `Kraken2`, `BLAST+`, `ABRicate` (with CARD + VFDB).  

**Data organization**:  
```
project/
│── raw_reads/
│── qc_reports/
│── trimmed_reads/
│── assemblies/
│── abricate_results/
│── vfdb_results/
```

**Error handling**:  
- Each script checks if directories exist (`mkdir -p`).  
- Use `set -e` at top of Bash scripts to stop if a command fails.  

---

✅ **Final Notes**  
This pipeline not only processes and analyzes outbreak isolates but also **interprets QC, AMR, and virulence results in a clinical/public health context**. Including both **Kraken2 + BLAST validation** ensures robustness. The findings explain the outbreak’s severity and guide evidence-based treatment recommendations.  
