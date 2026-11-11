# Variant-annotation-with-ANNOVAR-hg38

Same steps as the `anotando_annovar.ipynb` notebook (hg19) in [annotating-with-ANNOVAR-using-the-human-genome-Hg19](https://github.com/KairaCristina/annotating-with-ANNOVAR-using-the-human-genome-Hg19), with the build switched to **hg38** and databases replaced by their hg38 counterpart when the original version does not exist for that build.

| hg19 database | hg38 database | Reason |
|---|---|---|
| `clinvar_20140211` | `clinvar_20140702` | `clinvar_20140211` is hg19-only |
| `intervar_20170202` | `intervar_20180118` | `intervar_20170202` is hg19-only |
| `refGene`, `exac03`, `avsnp150`, `dbnsfp30a`, `dbnsfp31a_interpro`, `dbnsfp42a`, `clinvar_20221231`, `gnomad211_exome`, `gnomad_genome`, `revel` | unchanged | available for hg38 |

> The input files must be in **hg38/GRCh38** coordinates. If they are in hg19, lift them over first (e.g. CrossMap or Picard LiftoverVcf).

---

## 1. Create the conda environment

ANNOVAR is a set of Perl scripts. Database downloads (`-downdb`) use `wget` and `gunzip`.

```bash
conda create -n annovar_hg38 -c conda-forge -c bioconda --strict-channel-priority perl wget gzip
conda activate annovar_hg38

```
## 2. Set the paths once

Run this block in every new terminal session. All later commands use these variables, so they work from any folder.

```bash
BASE_DIR=path/to/your/ANNOVAR                 # working folder
ANNOVAR_DIR=$BASE_DIR/annovar                 # ANNOVAR scripts (created in step 3)
DB_HG38=$BASE_DIR/humandb_hg38                # hg38 databases (downloaded in step 4)
IN_DIR=$BASE_DIR/input                        # input files (VCF / avinput)
OUT_DIR=$BASE_DIR/output_hg38                 # annotation results
LOG_DIR=$BASE_DIR/logs                        # log files

mkdir -p $BASE_DIR $DB_HG38 $IN_DIR $OUT_DIR $LOG_DIR
cd $BASE_DIR
```
###
```
mv annovar.latest.tar.gz $BASE_DIR
```

> Every command below uses **absolute paths** through these variables. Mixing absolute and relative `humandb` paths is the usual cause of `database file does not exist` errors.

## 3. Install ANNOVAR

ANNOVAR requires a free registration: <https://www.openbioinformatics.org/annovar/annovar_download_form.php>. Download `annovar.latest.tar.gz` from the link sent by email and place it in `$BASE_DIR`.

```bash
tar -xvf $BASE_DIR/annovar.latest.tar.gz -C $BASE_DIR

# Must list annotate_variation.pl, table_annovar.pl, convert2annovar.pl, ...
ls $ANNOVAR_DIR
```

## 4. Download the hg38 databases

> Check free disk space first with `df -h $BASE_DIR`. `dbnsfp42a` alone takes ~45 GB once uncompressed, and `gnomad_genome` and `dbnsfp30a` are also tens of GB.

```bash
# Gene models
perl $ANNOVAR_DIR/annotate_variation.pl -buildver hg38 -downdb -webfrom annovar refGene $DB_HG38/

# Filter-based databases (skips databases already present; progress messages go to the .err files)
for db in exac03 avsnp150 dbnsfp30a dbnsfp31a_interpro dbnsfp42a \
          clinvar_20140702 clinvar_20221231 gnomad211_exome gnomad_genome \
          intervar_20180118 revel; do
    if [[ -s $DB_HG38/hg38_${db}.txt ]]; then echo "${db}: present - skipping"; continue; fi
    echo "${db}: downloading"
    perl $ANNOVAR_DIR/annotate_variation.pl -buildver hg38 -downdb -webfrom annovar ${db} $DB_HG38/ \
        1> $LOG_DIR/${db}_download.log 2> $LOG_DIR/${db}_download.err
done

# Empty output = all downloads finished successfully
grep -L "Finished downloading" $LOG_DIR/*_download.err
ls -lh $DB_HG38
```

> The downloads take hours. On a remote server, run the loop inside `tmux`/`screen`, or wrap it in `nohup bash -c '...' &`, so it does not stop if the connection drops. Do **not** start the same download twice: both processes write to the same file and corrupt it.
>
> The UniProt step of the original notebook was removed: the UCSC `uniprot` table is no longer available for download. Protein domains come from `dbnsfp31a_interpro`.


## 5. Input file

Create an example VCF with nine well-known variants in hg38 coordinates. Or replace it with your own hg38 VCF to annotate real data.

```bash
cat << 'EOF' | tr -s ' ' '\t' > $IN_DIR/variants.vcf
##fileformat=VCFv4.2
##FORMAT=<ID=GT,Number=1,Type=String,Description="Genotype">
#CHROM POS ID REF ALT QUAL FILTER INFO FORMAT SAMPLE1
chr10 112998590 rs7903146 C T . PASS . GT 0/1
chr19 44908684 rs429358 T C . PASS . GT 0/1
chr19 44908822 rs7412 C T . PASS . GT 0/1
chr16 53786615 rs9939609 T A . PASS . GT 0/1
chr3 12351626 rs1801282 C G . PASS . GT 0/1
chr8 117172544 rs13266634 C T . PASS . GT 0/1
chr6 26092913 rs1800562 G A . PASS . GT 0/1
chr1 11796321 rs1801133 G A . PASS . GT 0/1
chr11 17388025 rs5219 T C . PASS . GT 0/1
EOF
```

## 6. Annotation — `variants.vcf`

```bash
perl $ANNOVAR_DIR/table_annovar.pl $IN_DIR/variants.vcf $DB_HG38/ \
    -buildver hg38 \
    -out $OUT_DIR/variants_vcf \
    -remove \
    -protocol refGene,clinvar_20140702,clinvar_20221231,dbnsfp42a,dbnsfp30a,dbnsfp31a_interpro,intervar_20180118,exac03,gnomad211_exome,avsnp150,revel,gnomad_genome \
    -operation g,f,f,f,f,f,f,f,f,f,f,f \
    -nastring . \
    --dot2underline \
    -vcfinput \
    1> $LOG_DIR/variants_vcf.log 2> $LOG_DIR/variants_vcf.err
```

Outputs in `$OUT_DIR`:

| File | Content |
|---|---|
| `variants_vcf.hg38_multianno.txt` | Annotated table (one row per variant); the `Otherinfo*` columns hold the original VCF line, including genotypes |
| `variants_vcf.hg38_multianno.vcf` | Input VCF with all annotations added to the INFO column |
| `variants_vcf.avinput` | Intermediate file: the VCF converted to ANNOVAR input format |

> The number of databases in `-protocol` must equal the number of entries in `-operation` (`g` for gene-based, `f` for filter-based). Here: 12 and 12.

## 7. Convert the VCF to ANNOVAR input format (`.txt`)

```bash
perl $ANNOVAR_DIR/convert2annovar.pl -format vcf4old $IN_DIR/variants.vcf \
    -outfile $IN_DIR/variants.txt \
    2> $LOG_DIR/convert_variants.err

cat $IN_DIR/variants.txt
```

## 8. Annotation — `variants.txt`

Same databases, **without** `-vcfinput`, because the input is now in ANNOVAR format.

```bash
perl $ANNOVAR_DIR/table_annovar.pl $IN_DIR/variants.txt $DB_HG38/ \
    -buildver hg38 \
    -out $OUT_DIR/variants_txt \
    -remove \
    -protocol refGene,clinvar_20140702,clinvar_20221231,dbnsfp42a,dbnsfp30a,dbnsfp31a_interpro,intervar_20180118,exac03,gnomad211_exome,avsnp150,revel,gnomad_genome \
    -operation g,f,f,f,f,f,f,f,f,f,f,f \
    -nastring . \
    --dot2underline \
    1> $LOG_DIR/variants_txt.log 2> $LOG_DIR/variants_txt.err
```

Output: `variants_txt.hg38_multianno.txt` (same annotation columns as step 6, without genotypes or `Otherinfo` columns).

## 9. Check the results

```bash
ls -lh $OUT_DIR

# Errors or warnings of the annotation runs
grep -iE "error|warning" $LOG_DIR/variants_*.err

# Main columns of the result
awk -F'\t' 'NR==1{for(i=1;i<=NF;i++) h[$i]=i}
            {print $1, $2, $h["Gene_refGene"], $h["ExonicFunc_refGene"], $h["avsnp150"], $h["CLNSIG"]}' \
    OFS='\t' $OUT_DIR/variants_vcf.hg38_multianno.txt
```

Expected for the example VCF: each `avsnp150` rsID matches the `ID` column of the input; *TCF7L2* (rs7903146) and *FTO* (rs9939609) are intronic (`.` in `ExonicFunc_refGene`); the other seven are `nonsynonymous SNV`; *HFE* rs1800562 is `Pathogenic` in ClinVar.

---

## Notes

- **`-vcfinput`** is used only with VCF input (step 6). Using it on an ANNOVAR-format `.txt` file fails with `invalid record found in VCF4 file (at least 8 tab-delimited fields expected)`.
- **`WRONG reference alleles` warning:** the input coordinates are not hg38. Check the build of the file before interpreting the results.
- **Logs:** ANNOVAR writes its progress messages (`NOTICE`) to stderr, so read the `.err` files.
- **Output names:** always give `-out` a file prefix (e.g. `$OUT_DIR/variants_vcf`), not only a folder; otherwise the outputs become hidden files and each run overwrites the previous one.
- **Redundant databases:** steps 6 and 8 keep both ClinVar versions (2014 and 2022) and both dbNSFP versions (3.0a and 4.2a), as in the original notebook. REVEL is already included in `dbnsfp42a`. `dbnsfp42c` is the commercial-use version of `dbnsfp42a` and is not needed for academic work.
- **Newer versions** are available for hg38 (`clinvar_20240917`, `avsnp151`, `gnomad40_genome`, `dbnsfp47a_interpro`); swap them into `-protocol` and the download loop if exact reproduction of the hg19 notebook is not required.
- Database list and descriptions: <https://annovar.openbioinformatics.org/en/latest/user-guide/download/>
