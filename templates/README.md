# strain_mapper

[![Nextflow](https://img.shields.io/badge/nextflow%20DSL2-%E2%89%A521.04.0-23aa62.svg?labelColor=000000)](https://www.nextflow.io/)
[![run with docker](https://img.shields.io/badge/run%20with-docker-0db7ed?labelColor=000000&logo=docker)](https://www.docker.com/)
[![run with singularity](https://img.shields.io/badge/run%20with-singularity-1d355c.svg?labelColor=000000)](https://sylabs.io/docs/)

## Introduction

**<PROJECT>** is a pipeline for <PURPOSE>.

## Pipeline summary

<PROJECT_SUMMARY>

All relevant intermediate files are currently published in process-specific directories within the supplied `--outdir` directory.

## Getting started

### Running on the farm (Sanger HPC clusters)

1. Load nextflow and singularity modules:
   ```bash
   module load nextflow ISG/singularity
   ```

2. Clone the repo:
   ```bash
   git clone --recurse-submodules git@gitlab.internal.sanger.ac.uk:sanger-pathogens/pipelines/<PROJECT>.git
   cd <PROJECT>
   ```

3. Start the pipeline  
   For example input, please see [Generating a manifest](#generating-a-manifest).

   Example:
   ```bash
   nextflow run . --input ./test_data/inputs/test_manifest.csv --reference ./test_data/ref/test_ref.fna --outdir my_output
   ```

   It is good practice to submit a dedicated job for the nextflow master process (use the `oversubscribed` queue):
   ```bash
   bsub -o output.o -e error.e -q oversubscribed -R "select[mem>4000] rusage[mem=4000]" -M4000 nextflow run . --input ./test_data/inputs/test_manifest.csv --reference ./test_data/ref/test_ref.fna --outdir my_output
   ```

   See [usage](#usage) for all available pipeline options.

4. Once your run has finished, check output in the `outdir` and clean up any intermediate files. To do this (assuming no other pipelines are running from the current working directory) run:

   ```bash
   rm -rf work .nextflow*
   ```

## Generating a manifest of reads

Manifests of reads supplied as an argument to `--manifest_of_reads` should be of of the following format:

```console
ID,R1,R2
test_id,./test_data/inputs/test_1.fastq.gz,./test_data/inputs/test_2.fastq.gz
```

Where column `ID` can be an arbitrary sample identifier, `R1` is a .fastq.gz file of forward reads, `R2` is the mate .fastq.gz file containing reverse reads. 

Scripts have been developed to generate manifests appropriate for this pipeline:

- To generate a manifest from a file of lane identifiers visible to `pf`, use [this script](./scripts/generate_manifest_from_lanes.sh).

- To generate a manifest from a file of custom .fastq.gz paths, use [this script](./scripts/generate_manifest.sh).

Please run `--help` on these scripts for more information on script usage.


## Usage

```console
<PIPELINE_HELP_MESSAGE>
```
## Default behaviour

<PIPELINE_DEFAULT_BEHAVIOUR>

## Output

<PIPELINE_OUTPUT_DESCRIPTION>


## Contributions and testing

Developer contributions to this pipeline will only be accepted if all pipeline tests pass. To check:

1. Make your changes.

2. Download the test data. A utility script is provided:

   ```
   python3 scripts/download_test_data.py
   ```

3. Install [`nf-test`](https://code.askimed.com/nf-test/installation/) (>=0.7.0) and run the tests:

   ```
   nf-test test tests/*.nf.test
   ```

   If running on Sanger HPC cluster, add the option `--profile sanger_local`.

4. Submit a PR.

## Credits

PaM Informatics team, Wellcome Sanger Institute.

## Support

For further information or help, don't hesitate to get in touch via https://jira.sanger.ac.uk/servicedesk/customer/portal/16 or [pam-informatics@sanger.ac.uk](mailto:pam-informatics@sanger.ac.uk).

## Citations

If you use strain_mapper for your analysis, please cite the following doi: <PLACEHOLDER_FOR_CITATION>

An extensive list of references for the tools used by the pipeline can be found in the [`CITATIONS.md`](CITATIONS.md) file.