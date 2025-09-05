# Configuring Our Pipeline with Custom Resources

This README explains how to configure PaM Nextflow pipelines with custom resources (`CPU`, `memory`, `queue`, and `runtime`).
Examples use the `generate_mags` pipeline, but the same principles apply to all pipelines.

---

## 1. Download the `resources_tweak.config` template

The template resource_tweak.config provides examples for adjusting process-specific resources in PaM pipelines. You can use it as a starting point for your own resource configuration. To download use the below command in your working directory.

```bash
wget https://raw.githubusercontent.com/sanger-pathogens/nextflow-commons/refs/heads/master/configs/resource_tweak.config 
```

## 2. Identify processes for each pipeline

Each Nextflow pipeline is built from processes (tasks). To customise resources, you first need to know which processes are used in your pipeline. There are two ways to find them:

### Using our gitlab repositories: 

1) Go to the relevant GitLab repository for your pipeline. ([pathogen pipelines gitlab page](https://gitlab.internal.sanger.ac.uk/sanger-pathogens/pipelines))
2) Open the `main.nf` file 
3) Look at the `IMPORT MODULES/SUBWORKFLOWS` section. This lists all modules and subworkflows that are imported.

    example from generate_mags: 

    ```bash
    /*
    ========================================================================================
        IMPORT MODULES/SUBWORKFLOWS
    ========================================================================================
    */

    //
    // MODULES
    //
    include { ASSEMBLY; 
            BINNING;
            BIN_REFINEMENT;
            REASSEMBLE_BINS                } from './modules/metawrap.nf'
    include { CLEANUP_ASSEMBLY; 
            CLEANUP_BINNING; 
            CLEANUP_BIN_REFINEMENT; 
            CLEANUP_REFINEMENT_REASSEMBLY;
            CLEANUP_TRIMMED_FASTQ_FILES    } from './modules/cleanup/cleanup.nf'

    include { MIXED_INPUT                    } from './assorted-sub-workflows/mixed_input/mixed_input.nf'
    //
    // SUBWORKFLOWS
    //
    include { METAWRAP_QC                    } from './subworkflows/metawrap_qc.nf'

    /*
    ```

4) Navigate to the imported files (e.g. `./modules/metawrap.nf`) to find where the processes are defined.
    * Modules contain process definitions directly.
    * Subworkflows may import additional modules, so follow those paths as needed.

    In the example above, the processes `ASSEMBLY`, `BINNING`, `BIN_REFINEMENT`, and `REASSEMBLE_BINS` are defined inside modules/metawrap.nf

### Using an execution trace from a previous run: 


1) Copy the file pathway to the execution trace file 
2) in the terminal execute the below code: 

```bash

awk '!seen[$4]++ {print $4}' /path/to/execution_trace.txt

METAWRAP_QC:TRIMGALORE
METAWRAP_QC:BMTAGGER
METAWRAP_QC:GET_HOST_READS
...
```

This will print a list of processes in the format `SUBWORKFLOW_NAME_1:....SUBWORKFLOW_NAME_N:PROCESS_NAME`.Here, `METAWRAP_QC:TRIMGALORE` means that the `TRIMGALORE` process is called inside the `METAWRAP_QC` subworkflow

## 3. Identify the default resource settings for the processes of interest

Each process in a Nextflow pipeline has default resource settings. These control how many CPUs, how much memory, which queue, and how long the job can run. You may see these resources defined in different ways depending on the pipeline.

In all examples we will submit `ASSEMBLY` to the `normal` queue requesting `32GB` of memory and having a maximum time of `8 hours` to run.

1. Directly inside the process:

Some processes specify their resources explicitly in the process block:
```bash

process ASSEMBLY {
    cpus = 8
    memory = 32GB
    queue = normal 
    time = 8.h
}
```
2. Using labels from nextflow-commons:

Many pipelines use labels instead of hard-coding resources within the process block. These labels map to definitions in the [nextflow-commons.config](https://github.com/sanger-pathogens/nextflow-commons/blob/master/configs/nextflow.config) (see *section 8 "Understanding nextflow-commons labels"* for examples of label resolution)
```bash

process ASSEMBLY {
    label 'cpu_8'
    label 'mem_32'
    label 'time_queue_from_normal'
}
```

3) Using nextflow.config (in the main pipeline repository)

```bash
process { 
    withName ASSEMBLY {
            cpus = 8
            memory = 32GB
            queue = normal 
            time = 8.h
    }
}
```

**Tip:** When checking defaults:
1. Look in the process definition (`.nf` file)
2. Check if labels are used, and trace them back to the [nextflow-commons.config](https://github.com/sanger-pathogens/nextflow-commons/blob/master/configs/nextflow.config)
3. Check `nextflow.config` in the main pipeline repository


## 4. Priority:

Nextflow applies resource settings according to priority, with the highest-priority value taking effect if the same parameter is defined in multiple places. This allows you to override resource parameters. 

### Priorities low -> high:

1) Process definition (`.nf` file)
2) nextflow.config in the pipeline repository
3) Custom config passed with `-c` (e.g. resource_tweak.config) – highest priority and applied last.

## 5. How to override the resource requirements for your process of interest:

1) Edit the resource_tweak.config file by adding a process block with the exact name of the process as defined in the .nf file 
2) Specify the resource parameters you want to override.

Example:

```bash
process { 
    withName PROCESS_NAME {
        cpus = 8
        memory = 32GB
        queue = normal 
        time = 8 hr
    }
    withName PROCESS_NAME2 {
        cpus = 4
        memory = 20GB
        queue = normal 
        time = 8.h
    }
}
```
3) When running the pipeline add to the `nextflow run` command: `-c path/to/resource_tweak.config` to use your config

**What to watch out for :**

* We recommend including all key parameters: `cpus`, `memory`, `queue`, and `time` to ensure your job runs as you expect.
* If a parameter is omitted, it will inherit its value from a lower-priority source (`nextflow.config` > process defaults).
* Ensure the combination of `cpus`, `memory`, `queue`, and `time` is valid according to farm rules. See Section 6 for potential resource conflicts and consult the ISG farm documentation for further details.

**Tips:**

* To have two or more processes inherit the same resources, use the syntax: 
``` 'PROCESS1|PROCESS2' ```.

## 5. Resource conflicts:

When configuring resources, allocations must comply with ISG farm rules to avoid job submission errors. Always ensure you are up to date with the [ISG farm documentation](https://ssg-confluence.internal.sanger.ac.uk/spaces/FARM/pages/101360547/Farm+documentation), as normal LSF job submission rules apply.

### Memory and queue conflicts:

Jobs cannot request memory < 196 GB and submit to the hugemem queue.
Jobs cannot request memory < 745 GB and submit to the teramem queue.

### Label and parameter conflicts with examples:

If a parameter is not specified in `resource_tweak.config` for a process, the value will be inherited from `nextflow.config` as the next default, and then from the process-level defaults if not specified there. 

### Process script conflicts: 

Some scripts within processes may include resource arguments that conflict with your specified parameters. For example, in the `Jellyfish Generator` pipeline, the process `JELLY_GEN` has `memory` hard-coded to `50 GB` and `CPUs` set to `8`:

```bash
jellyfish count /dev/fd/0 -m 50 -s 100M -o ${db_name} -t 8 -C
```
To override this behavior, you would need to clone the pipeline and manually edit the command inside the process script.

## 6. Examples of overriding resources: 

The below section will show examples of how to overwrite the resource parameters and retry straties. 

All examples bellow will follow the ASSEMBLY process from generate mags.

**Process:**
```bash

process ASSEMBLY {
    label 'cpu_8'
    label 'mem_32'
    label 'time_queue_from_normal'
}
```
**Example 1:** Override all parameters

Using resource_tweak.config to override all resources at the process level:
```bash
process {
    withName: 'ASSEMBLY' {
        cpus = 4
        memory = 50.GB
        queue = 'long'
        time = 48.h
    }
}
```
**Result:** `ASSEMBLY` will run with `4 CPUs`, `50 GB memory`, on the `long queue`, for up to `48 hours`.`


**Example 2:** Override only some parameters

Using resource_tweak.config to some resources at the process level:
```bash
process {
    withName: 'ASSEMBLY' {
        memory = 50.GB
        queue = 'long'
        time = 48.h
    }
}
```
**Result:** CPUs remain from the process label (`cpu_8` → `8 CPUs`).`memory`, `queue`, and `time` are overridden.

**Example 3:** Partial override with label fallback

```bash
process {
    withName: 'ASSEMBLY' {
        memory = 50.GB
        queue = 'long'
    }
}
```
**Result:** Even though `queue = long` has been supplied, because `time ` parameter has not been specified, the default label `'time_queue_from_normal'` from the `nextflow-commons.config`will be applied. This label overrides the queue setting, and the job will be submitted to the normal queue. Refer to *Section 8 "Understanding nextflow-commons labels"*, for details.

**Example 4:** Memory resource escalation

```bash
process {
    withName: 'ASSEMBLY' {
        cpus    = 8
        memory  = { check_max( escalate_exp( 64.GB, task, 2 ), 'memory' ) }
        queue = 'long'
        time    = 48.h
    }
}
```
Ensure to add the folowing functions copied from the `nextflow-commons.config` to the `resource_tweak config`: 
1) `def check_max`
2) `def escalate_exp`

**Result:** The memory resource will start at `64 GB` on the first attempt.
On each retry (default 2 retries), the memory will double (`64` → `128` → `256 GB`, etc.) depending on the multiplier (in this case 2) .
Check_max ensures that the escalated memory does not exceed farm/system limits.

## 7. Recommended resource combinations:

Resource maximums by default are: 

* `max_memory = 2.9.TB`
* `max_cpus = 256`
* `max_time = 720.h`

Always request resources sensibly according to [ISG farm documentation](https://ssg-confluence.internal.sanger.ac.uk/spaces/FARM/pages/101360547/Farm+documentation), and if in doubt please raise a ticket with PaM Info or ISG.

For detailed memory limits and runtime restrictions by queue, use:
```bash
bqueues -l queue_name
```

### Recommended resource combinations:

Note: The memory recommendations for each job are based on current ISG documentation but have not received official ISG approval.

| Queue       | Memory               | Runtime Limit |
|-------------|----------------------|---------------|
| **small**   | < 256 GB             | ≤ 30 min      |
| **normal**  | < 256 GB             | ≤ 12 hours    |
| **long**    | < 256 GB             | ≤ 48 hours    |
| **week**    | < 256 GB             | ≤ 168 hours   |
| **hugemem** | 256–745 GB           | ≤ 360 hours   |
| **teramem** | > 745 GB             | ≤ 360 hours   |
| **basement**| < 256 GB             | ≤ 720 hours   |


## 8. Understanding nextflow-commons labels: 

Labels in Nextflow provide a central mechanism to define and manage resources for processes. To check what a label resolves to, refer to the [nextflow-commons.config](https://github.com/sanger-pathogens/nextflow-commons/blob/master/configs/nextflow.config). 

### Examples of label resolution

label `'cpu_8'`: 

```bash
    withLabel:cpu_8 {
        cpus   = { check_max( 8, 'cpus' ) }
    }
```
This resolves to 8 cpus.

label `'mem_32'`:

```bash
    withLabel:mem_32 {
        memory = { check_max( escalate_exp( 32.GB, task, 2 ), 'memory' ) }
    }
```
Resolves to `32 GB memory`.
Enables retries, escalating exponentially: `32 GB` → `64 GB` → `128 GB`.

label `'time_30m'`:

```bash
    withLabel:time_30m {
        time   = { check_max( escalate_linear( 30.m, task ), 'time' ) }
    }
```
Resolves to `30 minutes`.
Enables retries with linear escalation: `30 m` → `60 m` → `90 m`, etc.

label `'time_queue_from_normal'`:

```bash
    withLabel:time_queue_from_normal {
        time = { check_max( escalate_queue_time( 'normal', task ), 'time' ) }
    }
```
Resolves to `12 hours` initially.
If the task exceeds the maximum runtime, the time and queue are updated during retries:
* `12 h` → `normal queue`
* `48 h` → `long queue`
* `7 d` → `week queue`
* `30 d` → `basement queue`

Note: Even if you manually supply a queue, this label can override it during escalation so be careful.


**Tips:** 

* Verify label definitions to understand resource allocations and retry behavior as labels can override manually specified parameters.