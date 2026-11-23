# Vertebrate 12S reference database for PRONAME

PRONAME's built-in databases are bacterial, so 12S vertebrate reads come back `Unassigned`.
These steps build a vertebrate 12S database from NCBI GenBank with
[RESCRIPt](https://github.com/bokulich-lab/RESCRIPt) and point PRONAME's taxonomy step at it.

Based on the RESCRIPt tutorial
[Using RESCRIPt to compile sequence databases and taxonomy classifiers from NCBI GenBank](https://forum.qiime2.org/t/using-rescript-to-compile-sequence-databases-and-taxonomy-classifiers-from-ncbi-genbank/15947).
From that tutorial you only need the download step (`get-ncbi-data`). Its later steps train a
classifier, which PRONAME doesn't use: PRONAME uses BLAST, so it needs two files instead:

- a **BLAST database** of the reference sequences (PRONAME's `--db`)
- a **taxonomy file**: one line per reference sequence, its ID, a tab, then its lineage (PRONAME's `--reftax`)

## Steps

Run these in an SCC OnDemand desktop terminal.

**1. Make a folder for the database and load RESCRIPt**

```bash
mkdir -p /projectnb/vervmb/mfinn/vert12S_db
cd /projectnb/vervmb/mfinn/vert12S_db
module load miniconda qiime2/2025.4.0
conda activate /share/pkg.8/qiime2/2025.4.0/install-amplicon
```

**2. Download the sequences and their taxonomy from NCBI**

```bash
qiime rescript get-ncbi-data \
  --p-query 'Vertebrata[Organism] AND 12S[Title] AND 800:20000[SLEN]' \
  --o-sequences ncbi-seqs.qza \
  --o-taxonomy ncbi-taxonomy.qza
```

This fetches every vertebrate 12S record of 800 bp or more (48,255 on 2026-11-23).

`--p-query` is a normal [NCBI Nucleotide search](https://www.ncbi.nlm.nih.gov/nuccore). Try it on the
website first to see how many sequences it finds. The one above is an example; see
"Choosing the search" below.

**3. Convert to the two files PRONAME needs**

```bash
qiime tools export --input-path ncbi-seqs.qza --output-path exported
qiime tools export --input-path ncbi-taxonomy.qza --output-path exported
makeblastdb -in exported/dna-sequences.fasta -dbtype nucl -out blastdb/vert12S
tail -n +2 exported/taxonomy.tsv > vert12S_taxonomy.tsv
```

The last line removes the header line, which PRONAME doesn't expect.

**4. Point PRONAME at the new database**

In your `proname-scc/examples/ready-to-run/steps/taxonomy.sh`, change the `--db` and `--reftax` lines to:

```bash
  --db /projectnb/vervmb/mfinn/vert12S_db/blastdb/vert12S \
  --reftax /projectnb/vervmb/mfinn/vert12S_db/vert12S_taxonomy.tsv \
```

**5. Rerun only the taxonomy step**

```bash
cd /projectnb/vervmb/mfinn/proname-scc/examples/ready-to-run
bash submit.sh taxonomy
```

Results go to the same `WORKDIR` as before (`taxonomy_assay1_blastn.tsv`, `taxa_barplot_assay1_blastn.qzv`).

## Choosing the search

- **Sequence length matters.** PRONAME only accepts a BLAST match that covers at least 80% of your
  read (`--qcover 80`). Your reads are about 900–1,200 bp, so a short reference (for example a
  ~170 bp MiFish 12S fragment) can never cover 80% of one, and the read stays `Unassigned`. That is
  why the example search keeps only records of 800 bp or more (`800:20000[SLEN]`).
  In a test on the SCC, a 1,077 bp sequence matched 169 bp references of its own species at 100%
  identity, but they covered only 16% of it, so PRONAME left it `Unassigned`.
- More than half of the vertebrate 12S records on NCBI are shorter than 800 bp
  (counts on 2026-11-23: 48,255 records of 800 bp or more; 61,623 of 100–799 bp).
- To narrow the search to a group, change `Vertebrata` (for example `Actinopteri[Organism]` for
  ray-finned fishes).

## Run time

Measured on the SCC (2026-11-23), download step only:

| Sequences | Time |
|---|---|
| 995 | 45 s |
| 5,629 | 1 min 8 s |

About 1.4 GB of memory. NCBI asks that large downloads (RESCRIPt's guide: more than 125,000
sequences) run on weekends or between 9 pm and 5 am Eastern time.
