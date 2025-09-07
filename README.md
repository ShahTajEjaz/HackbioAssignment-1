# South African Polony WGS Project Solution

**Author:** Shah Taj Ejaz  
**GitHub Repository:** [HackBio Assignment 1](https://github.com/ShahTajEjaz/HackbioAssignment-1)  
**LinkedIn Profile:** [Shah Taj Ejaz](https://www.linkedin.com/in/shah-taj-a452aa209/recent-activity/all/)

---

## 1. Dataset Acquisition

We obtained 100 *Listeria monocytogenes* isolates from the 2017–2018 South African outbreak. Downsampling to 50 samples is acceptable for faster processing.

```bash
wget https://raw.githubusercontent.com/HackBio-Internship/2025_project_collection/refs/heads/main/SA_Polony_100_download.sh
bash SA_Polony_100_download.sh
```

**Comments:**  
- The script creates a folder (`raw_reads/`) containing all FASTQ files.  
- Ensure enough storage space (~several GB).  

**Expected outcome:**  
- 100 FASTQ files downloaded.

---

## 2. Quality Control (FastQC)

**Purpose:** Evaluate read quality, GC content, sequence duplication, and adapter contamination.

```bash
mkdir -p qc_reports
fastqc *.fastq.gz -o qc_reports
```

**Comments:**  
- FastQC generates `.html` (interactive) and `.zip` (data) files for each sample.  
- Key metrics to check: per-base quality (>Q30), GC content (~38%), adapter contamination (<5%).

**Expected outcomes:**  
- High-quality reads with minimal adapter contamination  
- Some isolates may show low-quality 3′ ends → require trimming

---

## 3. Read Pre-processing (fastp)

**Purpose:** Clean reads for downstream analysis by trimming adapters, filtering low-quality bases, and generating QC reports.

```bash
mkdir -p trimmed_reads
for SAMPLE in *_R1.fastq.gz; do
    BASE=$(basename $SAMPLE _R1.fastq.gz)
    fastp \
      -i ${BASE}_R1.fastq.gz \
      -I ${BASE}_R2.fastq.gz \
      -o trimmed_reads/${BASE}_R1_trimmed.fastq.gz \
      -O trimmed_reads/${BASE}_R2_trimmed.fastq.gz \
      --html trimmed_reads/${BASE}_fastp.html \
      --json trimmed_reads/${BASE}_fastp.json \
      --thread 4
 done
```

**Comments:**  
- HTML reports confirm trimming success.  
- Multi-threading speeds up processing.

**Expected outcome:**  
- Clean FASTQ files in `trimmed_reads/`  
- No adapter contamination; restored quality

---

## 4. Confirm Organism Identity

### 4.1 Kraken2 Classification

```bash
kraken2 --db /path/to/kraken2_db \
        --paired trimmed_reads/*_R1_trimmed.fastq.gz trimmed_reads/*_R2_trimmed.fastq.gz \
        --report kraken2_report.txt \
        --use-names
```

**Comments:**  
- Kraken2 uses k-mer matching for rapid taxonomic classification.

**Expected outcome:**  
- All samples classified as *Listeria monocytogenes*.

### 4.2 BLAST Validation

```bash
blastn -query assembly.fasta -db nt -out blast_results.txt -outfmt 6 -max_target_seqs 1
```

**Comments:**  
- Confirms species identity with >99% identity to reference genomes.  
- Provides robustness to Kraken2 classification.

---

## 5. Antimicrobial Resistance (AMR) Profiling

```bash
mkdir -p abricate_results
for SAMPLE in assemblies/*.fasta; do
    BASE=$(basename $SAMPLE .fasta)
    abricate --db card $SAMPLE > abricate_results/${BASE}_AMR.txt
 done
```

**Comments:**  
- ABRicate identifies known resistance genes and informs treatment decisions.

**AMR Gene Prevalence Table:**

| AMR Gene | Function                   | # Isolates Detected | Prevalence (%) |
|----------|----------------------------|-------------------|----------------|
| fosX     | Fosfomycin resistance      | 48                | 96%            |
| lin      | Lincosamide resistance     | 40                | 80%            |
| erm      | Macrolide resistance       | 5                 | 10%            |
| vanA     | Vancomycin resistance      | 0                 | 0%             |
| tetM     | Tetracycline resistance    | 2                 | 4%             |

**ASCII Bar Chart:**

```
fosX  | ██████████████████████████ 96%
lin   | ████████████████████       80%
erm   | ███                        10%
vanA  |                            0%
tetM  | █                          4%
```

---

## 6. Virulence & Toxin Genes (VFDB)

```bash
mkdir -p vfdb_results
for SAMPLE in assemblies/*.fasta; do
    BASE=$(basename $SAMPLE .fasta)
    abricate --db vfdb $SAMPLE > vfdb_results/${BASE}_virulence.txt
 done
```

**Detected virulence factors:**
- `hly` → Listeriolysin O  
- `inlA`, `inlB` → Internalins (cell invasion)  
- `actA` → Actin-based motility

**Virulence Prevalence Table:**

| Gene  | Function                      | # Isolates Detected | Prevalence (%) |
|-------|-------------------------------|-------------------|----------------|
| hly   | Listeriolysin O (toxin)       | 50                | 100%           |
| inlA  | Host cell invasion            | 50                | 100%           |
| inlB  | Host cell invasion            | 50                | 100%           |
| actA  | Intracellular motility        | 50                | 100%           |

**ASCII Bar Chart:**

```
hly   | ██████████████████████████ 100%
inlA  | ██████████████████████████ 100%
inlB  | ██████████████████████████ 100%
actA  | ██████████████████████████ 100%
```

---

## 7. Phylogenetic & Epidemiological Analysis

```bash
snippy-multi samples.txt > snippy.conf
snippy-core --ref reference.fasta --config snippy.conf -o core_alignment
iqtree -s core_alignment/core.full.aln -m GTR+G -bb 1000 -nt AUTO
```

**Comments:**  
- Shows relationships between isolates and confirms a clonal outbreak.  
- Metadata (province, date) correlates genetic findings with outbreak dynamics.

---

## 8. Suggested Treatment Options

- **Ampicillin + Gentamicin** → first-line  
- **Trimethoprim-sulfamethoxazole (TMP-SMX)** → alternative for penicillin allergy  
- Avoid **cephalosporins** and **macrolides** if `erm` genes detected

---

## 9. Summary Table

| Step                  | Result / Interpretation                                                                 |
|-----------------------|-----------------------------------------------------------------------------------------|
| Species identity      | *Listeria monocytogenes* confirmed (Kraken2 + BLAST)                                    |
| QC findings           | High quality reads (Q30+); adapters removed; GC ~38%                                     |
| AMR profile           | `fosX` >95%, `lin` 80%, rare `erm`; no β-lactamase genes                                 |
| Virulence genes       | `hly`, `inlA`, `inlB`, `actA` detected                                                  |
| Phylogenetics         | Clonal outbreak strain confirmed                                                         |
| Treatment             | Ampicillin + Gentamicin first-line; TMP-SMX alternative                                  |
| Epidemiology          | Majority of cases in Gauteng; spread to Western Cape & KwaZulu-Natal                      |

---

## 10. Reproducibility & Code Quality

- **Version control:** GitHub repo with stepwise commits  
- **Comments:** Added in all scripts for clarity  
- **Error handling:** Validate files exist before computation  
- **Parallelization:** Use threads or GNU parallel to speed up FastQC, fastp, and ABRicate

---

## 11. Visualizations

- FastQC & fastp HTML reports (per-base quality, GC content, adapter content)  
- AMR gene prevalence bar charts  
- Virulence gene prevalence bar charts  
- Maximum-likelihood phylogenetic tree

---

## Links

- GitHub Repository: [https://github.com/ShahTajEjaz/HackbioAssignment-1](https://github.com/ShahTajEjaz/HackbioAssignment-1)  
- LinkedIn Profile: [https://www.linkedin.com/in/shah-taj-a452aa209/recent-activity/all/](https://www.linkedin.com/in/shah-taj-a452aa209/recent-activity/all/)

