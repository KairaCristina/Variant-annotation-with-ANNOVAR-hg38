# Variant-annotation-with-ANNOVAR-hg38
Same steps as the anotando_annovar.ipynb notebook (hg19) in github.com/KairaCristina/annotating-with-ANNOVAR-using-the-human-genome-Hg19, with the build switched to hg38 and databases replaced by their hg38 counterpart when the original version does not exist for that build.

| hg19 database | hg38 database | Reason |
|---|---|---|
| `clinvar_20140211` | `clinvar_20140702` | `clinvar_20140211` is hg19-only |
| `intervar_20170202` | `intervar_20180118` | `intervar_20170202` is hg19-only |
| `refGene`, `exac03`, `avsnp150`, `dbnsfp30a`, `dbnsfp31a_interpro`, `dbnsfp42a`, `clinvar_20221231`, `gnomad211_exome`, `gnomad_genome`, `revel` | unchanged | available for hg38 |

> The input files must be in **hg38/GRCh38** coordinates. If they are in hg19, lift them over first (e.g. CrossMap or Picard LiftoverVcf).

---

## 0. Set the paths once

Run this block in every new terminal session. All later commands use these variables, so they work from any folder.

```bash
BASE_DIR=/home/users/kcptomaz/ANNOVAR         # folder that holds annovar/, humandb_hg38/ and the input files
ANNOVAR_DIR=$BASE_DIR/annovar                 # ANNOVAR scripts
DB_HG38=$BASE_DIR/humandb_hg38                # existing hg38 databases (already ~180 GB)
OUT_DIR=$BASE_DIR/output_hg38
LOG_DIR=$BASE_DIR/logs

mkdir -p $OUT_DIR $LOG_DIR
cd $BASE_DIR

# Check
ls $ANNOVAR_DIR/table_annovar.pl
ls $DB_HG38
```

> Every command below uses **absolute paths** through these variables. Mixing absolute and relative `humandb` paths is the usual cause of `database file does not exist` errors.

## 1. Extract ANNOVAR

```bash
tar -xvf $BASE_DIR/annovar.latest.tar.gz -C $BASE_DIR
ls $ANNOVAR_DIR       # must list annotate_variation.pl, table_annovar.pl, ...
```

## 2. Download the hg38 databases

```bash
# Gene models
perl $ANNOVAR_DIR/annotate_variation.pl -buildver hg38 -downdb -webfrom annovar refGene $DB_HG38/

# Filter-based databases (progress messages go to the .err files)
for db in exac03 avsnp150 dbnsfp30a dbnsfp31a_interpro dbnsfp42a \
          clinvar_20140702 clinvar_20221231 gnomad211_exome gnomad_genome \
          intervar_20180118 revel; do
    perl $ANNOVAR_DIR/annotate_variation.pl -buildver hg38 -downdb -webfrom annovar ${db} $DB_HG38/ \
        1> $LOG_DIR/${db}_download.log 2> $LOG_DIR/${db}_download.err
done

# Empty output = all downloads finished successfully
grep -L "Finished downloading" $LOG_DIR/*_download.err
ls -lh $DB_HG38
```

> `dbnsfp42a`, `gnomad_genome` and `dbnsfp30a` are large (tens of GB each). Check free disk space first with `df -h $BASE_DIR`.
>
> The UniProt step of the original notebook was removed: the UCSC `uniprot` table is no longer available for download. Protein domains come from `dbnsfp31a_interpro`.

## 3. Check the input files

ANNOVAR accepts two input formats, and `-vcfinput` must be used **only** with VCF:

| Format | Example line | Flag |
|---|---|---|
| ANNOVAR avinput | `1  120378678  120378678  C  T` | *(no flag)* |
| VCF | standard VCF, ≥ 8 columns, header lines starting with `#` | `-vcfinput` |

```bash
# First data line of each input: 5 columns = avinput, >= 8 columns = VCF
for f in variantes.txt novo.txt variants_estrelas.txt variantes.vcf; do
    echo "== $f"; grep -v '^#' $BASE_DIR/$f | head -n 1
done
```

##  Annotation — `novo.txt` (all databases)

```bash
perl $ANNOVAR_DIR/table_annovar.pl $BASE_DIR/novo.txt $DB_HG38/ \
    -buildver hg38 \
    -out $OUT_DIR/novo \
    -remove \
    -protocol refGene,clinvar_20140702,dbnsfp42a,exac03,avsnp150,dbnsfp30a,clinvar_20221231,gnomad211_exome,intervar_20180118,revel,gnomad_genome \
    -operation g,f,f,f,f,f,f,f,f,f,f \
    -nastring . \
    --dot2underline \
    1> $LOG_DIR/novo.log 2> $LOG_DIR/novo.err
```

## 8. Annotation — `variantes.vcf` 

```bash
perl $ANNOVAR_DIR/table_annovar.pl $BASE_DIR/variantes.vcf $DB_HG38/ \
    -buildver hg38 \
    -out $OUT_DIR/variants_last_gwas_diabetes \
    -remove \
    -protocol refGene,clinvar_20140702,dbnsfp42a,exac03,avsnp150,dbnsfp30a,clinvar_20221231,gnomad211_exome,intervar_20180118,revel,gnomad_genome \
    -operation g,f,f,f,f,f,f,f,f,f,f \
    -nastring . \
    --dot2underline \
    -vcfinput \
    1> $LOG_DIR/variants_last_gwas_diabetes.log 2> $LOG_DIR/variants_last_gwas_diabetes.err
```
```
perl $ANNOVAR_DIR/table_annovar.pl $BASE_DIR/variantes.vcf $DB_HG38/ \
    -buildver hg38 \
    -out $OUT_DIR/variants_example \
    -remove \
    -protocol refGene,clinvar_20240917,dbnsfp42a,dbnsfp31a_interpro,exac03,avsnp151,gnomad211_exome,gnomad40_genome,intervar_20180118 \
    -operation g,f,f,f,f,f,f,f,f \
    -nastring . \
    --dot2underline \
    -vcfinput \
    1> $LOG_DIR/variants_example.log 2> $LOG_DIR/variants_example.err
```

## 9. Check the results

```bash
ls -lh $OUT_DIR

# Errors or warnings of any run
grep -iE "error|warning" $LOG_DIR/*.err | grep -v "_download.err"

# Region distribution of one result (column 6 = Func_refGene)
cut -f6 $OUT_DIR/novo.hg38_multianno.txt | sort | uniq -c | sort -rn
```

---

### Notes

- **`-vcfinput`** is used only in step 8 (`variantes.vcf`). The `.txt` inputs are in ANNOVAR avinput format; using `-vcfinput` on them fails with `invalid record found in VCF4 file (at least 8 tab-delimited fields expected)`. If step 3 shows that a `.txt` file is actually a VCF, add `-vcfinput` to its command.
- **`WRONG reference alleles` warning:** the input coordinates are not hg38. Check the build of the file before interpreting results.
- **Logs:** ANNOVAR writes its progress messages (`NOTICE`) to stderr, so read the `.err` files.
- **Output files:** named `<prefix>.hg38_multianno.txt` in `$OUT_DIR`, so they do not overwrite the previous hg19 results.
- **Redundant databases:** steps 6 and 8 keep both ClinVar versions (2014 and 2022) and both dbNSFP versions (3.0a and 4.2a), as in the original notebook. REVEL is already included in `dbnsfp42a`.
