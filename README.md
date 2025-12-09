# Biobakery-nextflow Metaomics Exhibition Day

by Kevin Bonham and Danielle Pinto

**TOC**

1. [Setup](#setup)
  - [Software](#software)
  - [Databases](#databases)
  - [Workflow](#workflow)
2. [Running the workflow]
  - Setting parameters
  - Setting options
  - Running
3. Outputs

## Setup

The [`biobakery-nextflow`](https://github.com/bonhamlab/biobakery-nextflow) workflows
generate taxonomic and functional profiles of metagenomic data
using 3 tools:

- `kneaddata` (QC and cleaning of sequencing reads)
- `metaphlan` (taxonomic profiles via marker gene alignment)
- `humann` (functional profiling via 2-tiered gene function search)

But installing the right software and databases can be a bit tricky.

### Software

For convenience, we've created a docker container
with all of the necessary software.
Checkout https://github.com/bonhamlab/biobakery-containers
for more info and some `environment.yaml` files
that you can use if you'd rather manage your own `conda` environments.

If using docker, open your shell
(on the VM, you can do this via SSH or in VS code)

```sh
docker pull kescobo/nextflow-biobakery:v4-v0.1
```

This will take a bit.

Than, tell the shell how to run the container

```sh
alias kneaddata="docker run -it --rm --volume "$PWD":"$PWD" kescobo/nextflow-biobakery:v4-v0.1 kneaddata"
alias metaphlan="docker run -it --rm --volume "$PWD":"$PWD" kescobo/nextflow-biobakery:v4-v0.1 metaphlan"
alias humann="docker run -it --rm --volume "$PWD":"$PWD" kescobo/nextflow-biobakery:v4-v0.1 humann"
```

These aliases will not persist between sessions.
If you want to make them permanent,
either add these aliases to your `~/.bashrc`, `~/.zshrc`, or `~/.config/fish/config.fish`,
or create small executables in somewhere accessible to your `$PATH`.
Eg, on the VMs, there is a folder `~/bin` directory.
Add the following files:

`kneaddata`:
```sh
#!/bin/bash
docker run -it --rm --volume "$PWD":"$PWD" kescobo/nextflow-biobakery:v4-v0.1 kneaddata "$@"
```

`metaphlan`:
```sh
#!/bin/bash
docker run -it --rm --volume "$PWD":"$PWD" kescobo/nextflow-biobakery:v4-v0.1 metaphlan "$@"
```

`humann`:
```sh
#!/bin/bash
docker run -it --rm --volume "$PWD":"$PWD" kescobo/nextflow-biobakery:v4-v0.1 humann "$@"
```

Then make them executable:

```
chmod +x ~/bin/kneaddata ~/bin/metaphlan ~/bin/humann
```

#### Dealing with the file system

Note the `--volume` arguments in all of the commands above.
All of the docker commands assume that paths are within the container itself,
which doesn't have your files and which you can't actually write to.

The setup above mounts your current working directory
with the same path in the container,
so if all of your actions take place in your current directory
or subdirectories, and you use absolute paths, everything will work.

If you need other directories (eg your databases are in a shared drive somewhere),
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
$ docker run -it --rm --volume "$PWD":"$PWD" kescobo/nextflow-biobakery:v4-v0.1 kneaddata_database --download human_genome bowtie2 $PWD/biobakery-databases/kneaddata

$ docker run -it --rm --volume "$PWD":"$PWD" kescobo/nextflow-biobakery:v4-v0.1 metaphlan --install --index mpa_vOct22_CHOCOPhlAnSGB_202403 --db_dir $PWD/biobakery-databases/metaphlan

$ docker run -it --rm --volume "$PWD":"$PWD" kescobo/nextflow-biobakery:v4-v0.1 humann_database --download chocophlan full biobakery-databases/humann/ --update-config no
$ docker run -it --rm --volume "$PWD":"$PWD" kescobo/nextflow-biobakery:v4-v0.1 humann_databases --download utility_mapping biobakery-databases/humann/ --update-config no
$ docker run -it --rm --volume "$PWD":"$PWD" kescobo/nextflow-biobakery:v4-v0.1 humann_databases --download uniref uniref90_ec_filtered_diamond biobakery-databases/humann/ --update-config no
```


#### Metaphlan on the VM

If you are using the provided VM,
A couple of steps are needed to make the metaphlan database work.

```sh
$ cd /vol/volume/reference_databases/
$ mkdir metaphlan
$ mv full_mapping_v4_alpha/mpa_vOct22_CHOCOPhlAnSGB_202403.tar metaphlan/
$ cd metaphlan
$ tar xvf mpa_vOct22_CHOCOPhlAnSGB_202403.tar
$ cd ..
$ cd mkdir humann
$ mv chocophlan.v4_alpha humann/chocophlan
$ mv uniref90_annotated_v4_alpha_ec_filtered humann/uniref
$ mv full_mapping_v4_alpha humann/utility_mapping
```

### Workflow

Download and install the workflow from github

```sh
$ cd /vol/volume/sessions
$ curl -LO https://github.com/BonhamLab/biobakery-nextflow/archive/refs/tags/v0.1.tar.gz
$ tar xvzf v0.1.tar.gz
$ cd biobakery-nextflow-0.1/
$ ls
README.md             metaphlanstart.nf  single-end-params.yaml  tuftshpc-params.yaml
aws-params.yaml       nextflow.config    template-params.yaml
engaging-params.yaml  nf-test.config     test
main.nf               processes          tests
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
trimmomatic_path: "/opt/conda/bin/trimmomatic"
metaphlan_db: "/vol/volume/reference_databases/metaphlan"
metaphlan_index: "mpa_vOct22_CHOCOPhlAnSGB_202403"
humann_db: "/vol/volume/reference_databases/metaphlan"
```


