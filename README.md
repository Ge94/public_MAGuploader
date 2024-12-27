# Public bins and MAGs uploader
Python script to upload bins and MAGs in fasta format to ENA (European Nucleotide Archive). This script generates xmls and manifests necessary for submission with webin-cli.

It takes as input one tsv (tab-separated values) table in the following format:

| genome_name | genome_path | accessions | assembly_software | binning_software | binning_parameters | stats_generation_software | completeness | contamination | genome_coverage | metagenome | co-assembly | broad_environment | local_environment | environmental_medium | rRNA_presence | taxonomy_lineage |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| ERR4647712_crispatus | path/to/ERR4647712.fa.gz | ERR4647712 | megahit_v1.2.9 | MGnify-genomes-generation-pipeline_v1.0.0 | default | CheckM2_v1.0.1 | 100 | 0.38 | 14.2 | chicken gut metagenome | False | chicken | gut | mucosa | True | d__Bacteria;p__Firmicutes;c__Bacilli;o__Lactobacillales;f__Lactobacillaceae;g__Lactobacillus;s__Lactobacillus crispatus |

With columns indicating:
  * _genome_name_: genome id (unique string identifier)
  * _accessions_: run(s) or assembly(ies) the genome was generated from (DRR/ERR/SRRxxxxxx for runs, DRZ/ERZ/SRZxxxxxx for assemblies). If the genome was generated by a co-assembly of multiple runs, separate them with a comma.
  * _assembly_software_: assemblerName_vX.X
  * _binning_software_: binnerName_vX.X
  * _binning_parameters_: binning parameters
  * _stats_generation_software_: software_vX.X
  * _completeness_: `float`
  * _contamination_: `float`
  * _rRNA_presence_: `True/False` if all among 5S, 16S, and 23S genes, and at least 18 tRNA genes, have been detected in the genome
  * _NCBI_lineage_: full NCBI lineage, either in tax ids (`integers`) or `strings`. Format: x;y;z;...
  * _metagenome_: needs to be listed in the taxonomy tree [here](<https://www.ebi.ac.uk/ena/browser/view/408169?show=tax-tree>) (you might need to press "Tax tree - Show" in the right most section of the page)
  * _co-assembly_: `True/False`, whether the genome was generated from a co-assembly. N.B. the script only supports co-assemblies generated from the same project.
  * _genome_coverage_ : genome coverage against raw reads
  * _genome_path_: path to genome to upload (already compressed)
  * _broad_environment_: `string` (explanation following)
  * _local_environment_: `string` (explanation following)
  * _environmental_medium_: `string` (explanation following)

According to ENA checklist's guidelines, 'broad_environment' describes the broad ecological context of a sample - desert, taiga, coral reef, ... 'local_environment' is more local - lake, harbour, cliff, ... 'environmental_medium' is either the material displaced by the sample, or the one in which the sample was embedded prior to the sampling event - air, soil, water, ...
For host-associated metagenomic samples, the three variables can be defined similarly to the following example for the chicken gut metagenome: "chicken digestive system", "digestive tube", "caecum". More information can be found at [ERC000050](<https://www.ebi.ac.uk/ena/browser/view/ERC000050>) for bins and [ERC000047](<https://www.ebi.ac.uk/ena/browser/view/ERC000047>) for MAGs under field names "broad-scale environmental context", "local environmental context", "environmental medium"

Another example can be found [here](examples/input_example.tsv)

### Warnings

Raw-read runs from which genomes were generated should already be available on the INSDC (ENA by EBI, GenBank by NCBI, or DDBJ), hence at least one DRR|ERR|SRR accession should be available for every genome to be uploaded. Assembly accessions (ERZ|SRZ|DRZ) are also supported.

If uploading TPA (Third PArty) genomes, you will need to contact [ENA support](<https://www.ebi.ac.uk/ena/browser/support>) before using the script. They will provide instructions on how to correctly register a TPA project where to submit your genomes. If both TPA and non-TPA genomes need to be uploaded, please divide them in two batches and use the `--tpa` flag only with TPA genomes.

Files to be uploaded will need to be compressed (e.g. already in .gz format).

No more than 5000 genomes can be submitted at the same time.

## Register samples and generate pre-upload files
The script needs `python`, `pandas`, `requests`, and `ena-webin-cli` to run. We provide a yaml file for the generation of a conda environment:

```bash
# Create environment and install requirements
conda env create -f requirements.yml
conda activate genome_uploader
```

You can generate pre-upload files with:

```bash
python genomeuploader/genome_upload.py -u UPLOAD_STUDY --genome_info METADATA_FILE (--mags | --bins) --webin WEBIN_ID --password PASSWORD --centre_name CENTRE_NAME [--out] [--force] [--live] [--tpa]
```

where
  * `-u UPLOAD_STUDY`: study accession for genomes upload to ENA (in format ERPxxxxxx or PRJEBxxxxxx)
  * `---genome_info METADATA_FILE` : genomes metadata file in tsv format
  * `-m, --mags, --b, --bins`: select for bin or MAG upload. If in doubt, look at [their definition according to ENA](<https://ena-docs.readthedocs.io/en/latest/submit/assembly/metagenome.html>)
  * `--out`: output folder (default: working directory)
  * `--force`: forces reset of sample xmls generation
  * `--live`: registers genomes on ENA's live server. Omitting this option allows to validate samples beforehand (it will need the `-test` option in the upload command for the test submission to work)
  * `--webin WEBIN_ID`: webin id (format: Webin_XXXXX)
  * `--password PASSWORD`: webin password
  * `--centre_name CENTRE_NAME`: name of the centre generating and uploading genomes
  * `--tpa`: if uploading TPA (Third PArty) generated genomes

It is recommended to validate your genomes in test mode (i.e. without `--live` in the registration step and with `-test` during the upload) before attempting the final upload. Launching the registration in test mode will add a timestamp to the genome name to allow multiple executions of the test process.

Sample xmls won't be regenerated automatically if a previous xml already exists. If any metadata or value in the tsv table changes, `--force` will allow xml regeneration.

### Produced files:
The script produces the following files and folders:
```bash
bin_upload/MAG_upload
├── manifests
│    └── ...
├── ENA_backup.json                 # backup file to prevent re-download of metadata from ENA. Regeneration can be forced with --force
├── genome_samples.xml              # xml generated to register samples on ENA before the upload
├── registered_bins/MAGs.tsv        # list of genomes registered on ENA in live mode - needed for manifest generation
├── registered_bins/MAGs_test.tsv   # list of genomes registered on ENA in test mode - needed for manifest generation
└── submission.xml                  # xml used for genome registration on ENA
```

## Upload genomes
Once manifest files are generated, it is necessary to use ENA's webin-cli resource to upload genomes.

To test your submission (i.e. you registered your samples without the `--live` option with genome_upload.py), add the `-test` argument.

A live execution example within this repo is the following:
```bash
ena-webin-cli \
  -context=genome \
  -manifest=ERR123456_bin.1.manifest \
  -userName="Webin-XXX" \
  -password="YYY" \
  -submit
```

More information on ENA's webin-cli can be found [here](<https://ena-docs.readthedocs.io/en/latest/submit/general-guide/webin-cli.html>).
