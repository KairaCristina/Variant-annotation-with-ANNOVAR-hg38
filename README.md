# Variant-annotation-with-ANNOVAR-hg38
Same steps as the anotando_annovar.ipynb notebook (hg19) in github.com/KairaCristina/annotating-with-ANNOVAR-using-the-human-genome-Hg19, with the build switched to hg38 and databases replaced by their hg38 counterpart when the original version does not exist for that build.

| hg19 database | hg38 database | Reason |
|---|---|---|
| `clinvar_20140211` | `clinvar_20140702` | `clinvar_20140211` is hg19-only; `20140702` is the oldest version available for hg38 |
| `intervar_20170202` | `intervar_20180118` | `intervar_20170202` is hg19-only |
| `refGene`, `exac03`, `avsnp150`, `dbnsfp30a`, `dbnsfp31a_interpro`, `dbnsfp42a`, `clinvar_20221231`, `gnomad211_exome`, `gnomad_genome`, `revel` | unchanged | already available for hg38 |

> The input files (`variantes.txt`, `novo.txt`, `variants_estrelas.txt`, `variantes.vcf`) must be in **hg38/GRCh38** coordinates. If they are in hg19, lift them over first (e.g. CrossMap or Picard LiftoverVcf).

---

## 1. Extract ANNOVAR

```bash
tar -xvf /home/kaira/Desktop/annovar/annovar.latest.tar.gz
```

## 2. Download the hg38 databases

```bash
perl /home/kaira/Desktop/annovar/annovar/annotate_variation.pl -buildver hg38 -downdb -webfrom annovar refGene humandb/
```

## 3. Download the remaining databases (hg38)

```bash
# hg38 - exac03
# perl /home/kaira/Desktop/annovar/annovar/annotate_variation.pl -buildver hg38 -downdb -webfrom annovar exac03 humandb/

# hg38 - avsnp150
# perl annovar/annotate_variation.pl -buildver hg38 -downdb -webfrom annovar avsnp150 annovar/humandb/ \
# 1>logs/avsnp150_down.txt 2>logs/avsnp150_down_errors.txt
# perl /home/kaira/Desktop/annovar/annovar/annotate_variation.pl -buildver hg38 -downdb -webfrom annovar avsnp150 humandb/

# hg38 - dbnsfp30a
# perl annovar/annotate_variation.pl -buildver hg38 -downdb -webfrom annovar dbnsfp30a annovar/humandb/ \
# 1>logs/dbnsfp30a_down.txt 2>logs/dbnsfp30a_down_errors.txt
# perl /home/kaira/Desktop/annovar/annovar/annotate_variation.pl -buildver hg38 -downdb -webfrom annovar dbnsfp30a humandb/

# hg38 - clinvar_20221231
# perl annovar/annotate_variation.pl -buildver hg38 -downdb -webfrom annovar clinvar_20221231 annovar/humandb/ \
# 1>logs/clinvar_20221231_down.txt 2>logs/clinvar_20221231_down_errors.txt
# perl /home/kaira/Desktop/annovar/annovar/annotate_variation.pl -buildver hg38 -downdb -webfrom annovar clinvar_20221231 humandb/

# hg38 - gnomad211_exome
# perl annovar/annotate_variation.pl -buildver hg38 -downdb -webfrom annovar gnomad211_exome annovar/humandb/ \
# 1>logs/gnomad211_exome_down.txt 2>logs/gnomad211_exome_down_errors.txt
# perl /home/kaira/Desktop/annovar/annovar/annotate_variation.pl -buildver hg38 -downdb -webfrom annovar gnomad211_exome humandb/

# hg38 - intervar_20180118 (intervar_20170202 is hg19-only)
# perl annovar/annotate_variation.pl -buildver hg38 -downdb -webfrom annovar intervar_20180118 annovar/humandb/ \
# 1>logs/intervar_20180118_down.txt 2>logs/intervar_20180118_down_errors.txt
perl /home/kaira/Desktop/annovar/annovar/annotate_variation.pl -buildver hg38 -downdb -webfrom annovar intervar_20180118 humandb/

# hg38 - revel
# perl annovar/annotate_variation.pl -buildver hg38 -downdb -webfrom annovar revel annovar/humandb/ \
# 1>logs/revel_down.txt 2>logs/revel_down_errors.txt
# perl /home/kaira/Desktop/annovar/annovar/annotate_variation.pl -buildver hg38 -downdb -webfrom annovar revel humandb/
```

## 4. ClinVar (older version)

```bash
# clinvar_20140211 is hg19-only; clinvar_20140702 is the oldest version available for hg38
perl /home/kaira/Desktop/annovar/annovar/annotate_variation.pl -buildver hg38 -downdb -webfrom annovar clinvar_20140702 humandb/
```

## 5. dbNSFP 4.2a

```bash
# run again
perl /home/kaira/Desktop/annovar/annovar/annotate_variation.pl -buildver hg38 -downdb -webfrom annovar dbnsfp42a humandb/
```

## 6. UniProt (UCSC)

```bash
perl /home/kaira/Desktop/annovar/annovar/annotate_variation.pl -buildver hg38 -downdb uniprot humandb/
```

## 7. gnomAD genome

```bash
perl /home/kaira/Desktop/annovar/annovar/annotate_variation.pl -buildver hg38 -downdb -webfrom annovar gnomad_genome humandb/
```

## 8. Annotation — `variantes.txt` (refGene, ClinVar, dbNSFP)

```bash
perl /home/kaira/Desktop/annovar/annovar/table_annovar.pl variantes.txt /home/kaira/Desktop/annovar/humandb -buildver hg38 -out results \
-protocol refGene,clinvar_20140702,dbnsfp42a -operation g,f,f -nastring . -vcfinput
```

## 9. Annotation — `variantes.txt` (gnomAD, ClinVar, InterPro)

```bash
perl /home/kaira/Desktop/annovar/annovar/table_annovar.pl variantes.txt humandb/ \
    -buildver hg38 \
    -out file.avinput \
    -remove \
    -protocol refGene,gnomad211_exome,clinvar_20221231,dbnsfp31a_interpro,gnomad_genome \
    -operation g,f,f,f,f \
    -nastring "." \
    -vcfinput \
    --dot2underline
```

## 10. Annotation — `novo.txt` (all databases)

```bash
perl /home/kaira/Desktop/annovar/annovar/table_annovar.pl /home/kaira/Desktop/annovar/novo.txt humandb/ \
    -buildver hg38 \
    -out new \
    -remove \
    -protocol refGene,clinvar_20140702,dbnsfp42a,exac03,avsnp150,dbnsfp30a,clinvar_20221231,gnomad211_exome,intervar_20180118,revel,gnomad_genome \
    -operation g,f,f,f,f,f,f,f,f,f,f \
    -nastring "." \
    --dot2underline
```

## 11. Annotation — `variants_estrelas.txt`

```bash
perl /home/kaira/Desktop/annovar/annovar/table_annovar.pl /home/kaira/Desktop/annovar/variants_estrelas.txt humandb/ \
    -buildver hg38 \
    -out star_variants \
    -remove \
    -protocol refGene,gnomad211_exome,dbnsfp42a \
    -operation g,f,f \
    -nastring "." \
    --dot2underline
```

## 12. Annotation — `variantes.vcf` (diabetes GWAS, all databases)

```bash
perl /home/kaira/Desktop/annovar/annovar/table_annovar.pl /home/kaira/Desktop/annovar/variantes.vcf humandb/ \
    -buildver hg38 \
    -out variants_last_gwas_diabetes \
    -remove \
    -protocol refGene,clinvar_20140702,dbnsfp42a,exac03,avsnp150,dbnsfp30a,clinvar_20221231,gnomad211_exome,intervar_20180118,revel,gnomad_genome \
    -operation g,f,f,f,f,f,f,f,f,f,f \
    -nastring "." \
    --dot2underline \
    -vcfinput
```

---

### Notes

- **Commented-out downloads (step 3):** as in the original notebook, only InterVar is active. Uncomment the others (`exac03`, `avsnp150`, `dbnsfp30a`, `clinvar_20221231`, `gnomad211_exome`, `revel`) to download them for hg38 — steps 10 and 12 use all of them.
- **`dbnsfp31a_interpro` (step 9)** has no download step, also in the original. To download it: `perl /home/kaira/Desktop/annovar/annovar/annotate_variation.pl -buildver hg38 -downdb -webfrom annovar dbnsfp31a_interpro humandb/`
- **`humandb` path:** step 8 uses `/home/kaira/Desktop/annovar/humandb` while the others use the relative `humandb/`, as in the original. Run everything from `/home/kaira/Desktop/annovar/` so both point to the same folder.
- **Input file names** (`variantes.txt`, `novo.txt`, `variants_estrelas.txt`, `variantes.vcf`) were kept as-is because they refer to existing files; rename them on disk if you want English names there too.
- **Output files:** ANNOVAR names outputs by build (`*.hg38_multianno.txt`), so they will not overwrite the previous hg19 results.
