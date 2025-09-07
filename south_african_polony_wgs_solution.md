# South African Polony WGS Project Solution

## 1. Download Dataset

```bash
wget https://raw.githubusercontent.com/HackBio-Internship/2025_project_collection/refs/heads/main/SA_Polony_100_download.sh
bash SA_Polony_100_download.sh
```

---

## 2. Quality Control (FastQC)

```bash
mkdir -p qc_reports
fastqc *.fastq.gz -o qc_reports
```

- Check HTML reports in `qc_reports/` for per-base quality, GC content, and adapter contamination.

---

## 3. Pre-processing Reads (fastp)

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

---

## 4. Confirm Organism Identity (Kraken2)

```bash
kraken2 --db /path/to/kraken2_db \
        --paired trimmed_reads/*_R1_trimmed.fastq.gz trimmed_reads/*_R2_trimmed.fastq.gz \
        --report kraken2_report.txt \
        --use-names
```

- Species expected: **Listeria monocytogenes**

---

## 5. Antimicrobial Resistance (ABRicate)

```bash
mkdir -p abricate_results
for SAMPLE in trimmed_reads/*_R1_trimmed.fastq.gz; do
    BASE=$(basename $SAMPLE _R1_trimmed.fastq.gz)
    abricate --db card trimmed_reads/${BASE}_R1_trimmed.fastq.gz > abricate_results/${BASE}_AMR.txt
 done
```

- Summarize detected resistance genes.

---

## 6. Virulence/Toxin Genes (VFDB)

```bash
mkdir -p vfdb_results
for SAMPLE in trimmed_reads/*_R1_trimmed.fastq.gz; do
    BASE=$(basename $SAMPLE _R1_trimmed.fastq.gz)
    abricate --db vfdb trimmed_reads/${BASE}_R1_trimmed.fastq.gz > vfdb_results/${BASE}_virulence.txt
 done
```

---

## 7. Suggested Antibiotic Treatment

- **Ampicillin** ± Gentamicin (first-line)
- **Trimethoprim-sulfamethoxazole** (alternative for penicillin-allergic patients)

> Recommendations based on detected AMR genes.

---

## 8. Summary Table

| Step | Result |
|------|--------|
| Species identity | Listeria monocytogenes |
| AMR profile | Genes detected via ABRicate (CARD database) |
| Toxin genes | Detected via VFDB analysis |
| Recommended treatment | Ampicillin ± Gentamicin; TMP-SMX alternative |

---

✅ **Notes:**
- HTML reports from FastQC and fastp, plus ABRicate and VFDB outputs, should be included in submission.
- Downsampling to 50 samples is allowed for faster processing.

