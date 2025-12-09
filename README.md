# Biobakery-nextflow Metaomics Exhibition Day

by Kevin Bonham and Danielle Pinto

**TOC**

1. [Setup](#setup)
  1. [Software](#software)
  1. [Databases](#databases)
  1. [Workflow](#workflow)
1. [Running the workflow](#running-the-workflow)
  1. Setting parameters
  1. Setting options
  1. Running
1. Outputs

## Setup

The [`biobakery-nextflow`](https://github.com/bonhamlab/biobakery-nextflow) workflows
generate taxonomic and functional profiles of metagenomic data
using 3 tools:

- `kneaddata` (QC and cleaning of sequencing reads)
- `metaphlan` (taxonomic profiles via marker gene alignment)
- `humann` (functional profiling via 2-tiered gene function search)

But installing the right software and databases can be a bit tricky.

### Software

There are a few different ways to get the appropriate software.

#### Conda / Mamba

All of the software is available in bioconda.
Create a file called `evironment.yaml`
and install the required software.

```yaml
name: base
channels:
  - bioconda
  - conda-forge
  - biobakery
dependencies:
  - python=3.12
  - metaphlan=4.1
  - humann=4.0.0a1
  - kneaddata=0.12
  - procps-ng
```

Then build it 
(change to a different environment name if you don't want it
in your base conda environment) and activate:

```sh
mamba install -y -n base -f environment.yaml
mamba activate
```

The Java within the conda environment is not compatible with nextflow,
so you need to explicitly set it to use the system java.

```sh
export JAVA_HOME=/usr/lib/jvm
export JAVA_CMD=/usr/bin/java
```

#### Docker / Singularity

If you're comfortable with container workflows,
we've created a docker/singularity containers
if you're comfortable with that.

If using docker, open your shell
(on the VM, you can do this via SSH or in VS code)

```sh
docker pull kescobo/nextflow-biobakery:v4-v0.1
```

This will take a bit.

The workflow expects certain executables
to be available on your $PATH.
The easiest way to deal with this is to
create small executables in somewhere accessible to your `$PATH`.
Eg, on the VMs, there is a folder `~/bin` directory.
Add the following files:

`kneaddata`:
```sh
#!/bin/bash
docker run -it --rm --volume "/vol/volume/":"/vol/volume/" kescobo/nextflow-biobakery:v4-v0.1 kneaddata "$@"
```

`metaphlan`:
```sh
#!/bin/bash
docker run -it --rm --volume "/vol/volume/":"/vol/volume/" kescobo/nextflow-biobakery:v4-v0.1 metaphlan "$@"
```

`humann`:
```sh
#!/bin/bash
docker run -it --rm --volume "/vol/volume/":"/vol/volume/" kescobo/nextflow-biobakery:v4-v0.1 humann "$@"
```

Then make them executable:

```
chmod +x ~/bin/kneaddata ~/bin/metaphlan ~/bin/humann
```

#### Dealing with the file system

Note the `--volume` arguments in all of the commands above.
All of the docker commands assume that paths are within the container itself,
which doesn't have your files and which you can't actually write to.

The setup above mounts `/vol/volume` on the vm
with the same path in the container,
or subdirectories, and you use absolute paths,
so if you use absolute paths, everything will work.

If you need other directories (eg your databases are in a shared drive somewhere)
or you're using your home directory,
be sure to add `--volume` arguments for those paths as well.

### Databases

The docker container has the software,
but each of these tools also requires one or more databases.

> [!NOTE]
> If you are running on the VM,
> DO NOT take the following steps - the databases have already been downloaded
> and are available.
> However, the metaphlan database was a bit messed up,
> jump to [here](#metaphlan-on-the-vm).

Each tool comes with functionality to download their databases
(`kneaddata_database`, `metaphlan --install`, `humann_database`),
but all of these tools assume that they have access to your filesystem
(which the containers do not).
So I usually manage them separately.

Some of these databases are quite large,
so be prepared to wait

```sh
kneaddata_database --download human_genome bowtie2 $PWD/biobakery-databases/kneaddata

metaphlan --install --index mpa_vOct22_CHOCOPhlAnSGB_202403 --db_dir $PWD/biobakery-databases/metaphlan

humann_database --download chocophlan full biobakery-databases/humann/ --update-config no
humann_databases --download utility_mapping biobakery-databases/humann/ --update-config no
humann_databases --download uniref uniref90_ec_filtered_diamond biobakery-databases/humann/ --update-config no
```


#### Metaphlan on the VM

If you are using the provided VM,
A couple of steps are needed to make the metaphlan database work.

```sh
cd /vol/volume/reference_databases/
mkdir metaphlan
mv full_mapping_v4_alpha/mpa_vOct22_CHOCOPhlAnSGB_202403.tar metaphlan/
cd metaphlan
tar xvf mpa_vOct22_CHOCOPhlAnSGB_202403.tar
cd ..
cd mkdir humann
mv chocophlan.v4_alpha humann/chocophlan
mv uniref90_annotated_v4_alpha_ec_filtered humann/uniref
mv full_mapping_v4_alpha humann/utility_mapping
```

### Workflow

Download and install the workflow from github

```sh
cd /vol/volume/sessions
curl -LO https://github.com/BonhamLab/biobakery-nextflow/archive/refs/tags/v0.2.tar.gz
tar xvzf v0.2.tar.gz
cd biobakery-nextflow-0.2/
ls
```
```
README.md  metaphlanstart.nf  nf-test.config  template-params.yaml  tests
main.nf    nextflow.config    processes       test
```

## Running the Workflow

### Workflow parameters

Make a copy of the `template-params.yaml` file
called `params.yaml`.

If you're using the same files as the workshop,
the `paied_end`, `filepattern`, and `metaphlan_version` and 
`humann_version` parameters are already correct.
If you're using your own files, the `filepattern` may need to be tweaked.

Now, replace change the following keys to reflect the current environment.
On the VM, this should be:

```yaml
readsdir: "/vol/volume/sessions/BonhamLab_biobakery-nextflow/input"
outdir: "/vol/volume/sessions/BonhamLab_biobakery-nextflow/output"
human_genome: "/vol/volume/reference_databases/Homo_sapiens_hg39_T2T_Bowtie2_v0.1/"
metaphlan_db: "/vol/volume/reference_databases/metaphlan"
metaphlan_index: "mpa_vOct22_CHOCOPhlAnSGB_202403"
humann_db: "/vol/volume/reference_databases/humann"
```

### Run the workflow

> [!NOTE]
> Using the included data can take a long time
> because they're full samples.
> If you'd like to use smaller samples to see the workflow end-to-end,
> you can replace the files in `/vol/volume/sessions/BonhamLab_biobakery-nextflow/input`
> with the files in the `test/rawfastq` subdirectory of the repository.

```sh
nextflow run main.nf -profile standard -params-file params.yaml
```


