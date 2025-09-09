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

Each Nextflow pipeline is built from processes, which are applied to each input data item (task). To customise resources, you first need to know which processes are used in your pipeline. There are two ways to find them:

### Using our gitlab repositories: 

1) Go to the relevant git repository for your pipeline. ([PaM Informatics pipelines page on WSI private Gitlab](https://gitlab.internal.sanger.ac.uk/sanger-pathogens/pipelines) or [public Github page](https://github.com/sanger-pathogens))
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
    * Modules contain process definitions.
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

This will print a list of processes in the format `SUBWORKFLOW_NAME_1:....SUBWORKFLOW_NAME_N:PROCESS_NAME`. Here, `METAWRAP_QC:TRIMGALORE` means that the `TRIMGALORE` process is called inside the `METAWRAP_QC` subworkflow

## 3. Identify the default resource settings for the processes of interest

Each process in a Nextflow pipeline has default resource settings. These control how many CPUs, how much memory the task will be allowed to use, which queue the task job is submitted to (if executing pipeline on an HPC), and how long the submitted job can run. You may see these resources defined in different ways depending on the pipeline.   
Note that some resource specification are related or even redundant. For instance, specifying the HPC queue may constrain the maximum time a job can run, regardless of the max timespcified in the config; e.g. on WSI "farm" HPC, jobs submitted to the `normal` queue can't run longer than 12h.```

In all examples below the pipeline will submit `ASSEMBLY` task jobs to the `normal` queue requesting `32GB` of memory and having a maximum time of `8 hours` to run.

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

Many pipelines use labels instead of hard-coding resources within the process block. These labels map to definitions in the [nextflow-commons.config](nextflow.config) (see *section 8 "Understanding nextflow-commons labels"* for examples of label resolution)
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
    withName: ASSEMBLY {
            cpus = 8
            memory = 32GB
            queue = normal 
            time = 8.h
    }
}
```

**Tip:** When checking defaults:
1. Look in the process definition (`.nf` module file)
2. Check if labels are used, and trace them back to the [nextflow-commons.config](nextflow.config)
3. Check `nextflow.config` in the main pipeline repository


## 4. Priority:

Nextflow applies configuration i.e. set parameter values, including those pertaining to resource settings, according to the configuration source priority, with the highest-priority parameter value taking effect if the same parameter is defined in multiple places. This allows you to override default resource parameter settings. 

### Priorities low -> high:

1) Process definition (`.nf` file)
2) nextflow.config in the pipeline repository
3) Custom config passed with `-c` (e.g. resource_tweak.config)
4) parameters defined through the command-line interface (with `--` options) – highest priority and applied last

## 5. How to override the resource requirements for your process of interest:

1) Edit the `resource_tweak.config` file by adding a process block with the exact name of the process as defined in the `.nf` file 
2) Specify the resource parameters you want to override.

Example:

```bash
process { 
    withName: PROCESS_NAME {
        cpus = 8
        memory = 32GB
        queue = normal 
        time = 8 hr
    }
    withName: PROCESS_NAME2 {
        cpus = 4
        memory = 20GB
        queue = normal 
        time = 8.h
    }
}
```
3) When running the pipeline, append the following to the `nextflow run` command: `-c path/to/resource_tweak.config` to use your config file.

**What to watch out for :**

* We recommend including all key parameters: `cpus`, `memory`, `queue`, and `time` to ensure your job runs as you expect.
* If a parameter is omitted, it will inherit its value from a lower-priority source (`nextflow.config` > process defaults).
* Ensure the combination of `cpus`, `memory`, `queue`, and `time` is valid according to farm rules. See Section 6 for potential resource conflicts and consult the ISG farm documentation for further details.

**Tips:**

* To have two or more processes inherit the same resources, use the syntax: 
``` 'PROCESS1|PROCESS2' ```.

## 5. Resource conflicts:

When configuring resources for use on WSI "farm" HPC, allocations must comply with ISG farm rules to avoid job submission errors. Always ensure you are up to date with the [ISG documentation on the farm queues](https://ssg-confluence.internal.sanger.ac.uk/spaces/FARM/pages/101361122/What+are+the+different+queues+for), as normal LSF job submission rules apply.

### Memory and queue conflicts:

Jobs cannot request memory > 256 GB and submit to the normal, long, week and basement queues.
Jobs cannot request memory < 196 GB and submit to the hugemem queue.
Jobs cannot request memory < 745 GB and submit to the teramem queue.

### Label and parameter conflicts with examples:

If a parameter is not specified in `resource_tweak.config` for a process, the value will be inherited from `nextflow.config` as the next default, and then from the process-level defaults if not specified there. 
Note that the pipeline's `nextflow.config` may be importing other config files, for instance the [PaM Informatics common config](nextflow.config)

### Process script conflicts: 

Some scripts within processes may include resource arguments that conflict with your specified parameters. This may cause task failures due to excess resource use, or suboptimal runtime efficiency. For example, in the `Jellyfish Generator` pipeline, the `jellyfish` command in the process `JELLY_GEN` has its max memory use hard-coded to `50 GB` and thread count set to `8`:

```bash
jellyfish count /dev/fd/0 -m 50 -s 100M -o ${db_name} -t 8 -C
```
To override this behavior, you would need to clone the pipeline and manually edit the command inside the process script.

## 6. Examples of overriding resources: 

The below section will show examples of how to override the resource parameters and retry strategies. 

All examples bellow will follow the ASSEMBLY process from generate mags.

**Process, as defined in the `*.nf` module file:**
```bash

process ASSEMBLY {
    label 'cpu_8'
    label 'mem_32'
    label 'time_queue_from_normal'
    
    [...]
}
```
### **Example 1:** Override all resource parameters

Using `resource_tweak.config` to override all resources for a specific process:
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


### **Example 2:** Override only some parameters

Using `resource_tweak.config` to override some resources for a specific process:
```bash
process {
    withName: 'ASSEMBLY' {
        memory = 50.GB
        queue = 'long'
        time = 48.h
    }
}
```
**Result:** `cpu` task parameter value remain as set from the process label (`cpu_8` → 8 CPUs).`memory`, `queue`, and `time` task parameter values are overridden.

### **Example 3:** Partial override with label fallback

```bash
process {
    withName: 'ASSEMBLY' {
        memory = 50.GB
        queue = 'long'
    }
}
```
**Result:** Maximum task job runtime remains 12h. Why? Even though `queue = long` has been specified in the `resource_tweak.config` file, the `time ` parameter has not been modified there, and takes its value from the default process label `'time_queue_from_normal'`, which is interpreted as per the `nextflow-commons.config`. Applying this label overrides the queue setting, and the job will be submitted to the normal queue, and accordingly sets the task paramter `time` value to 12 . Refer to *Section 8 "Understanding nextflow-commons labels"*, for more details.

### **Example 4:** Changing memory resource escalation strategy

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
1) `def check_max { ... }`
2) `def escalate_exp { ... }`
3) `def escalate_linear { ... }`

**Result:** The memory resource requested for the task job will start at 64 GB on the first attempt.
On each retry (default 2 retries), the memory will then double (64 → 128 → 256 GB, etc.) depending on the multiplier (in this case 2) .
The `check_max` function ensures that the escalated memory does not exceed farm/system limits.

## 7. Recommended resource combinations:

Resource maximums by default are: 

* `max_memory = 2.9.TB`
* `max_cpus = 256`
* `max_time = 720.h`

Always request resources sensibly according to ISG farm documentation [here](https://ssg-confluence.internal.sanger.ac.uk/spaces/FARM/pages/101360547/Farm+documentation) and [there for more details on what queue to use](https://ssg-confluence.internal.sanger.ac.uk/spaces/FARM/pages/101361122/What+are+the+different+queues+for), and if in doubt please raise a ticket with PaM Info or ISG.

For detailed memory limits and runtime restrictions by queue, use:
```bash
bqueues -l queue_name
```

### Recommended resource combinations on WSI "farm" HPC:

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

Labels in Nextflow provide a central mechanism to define and manage resources for processes. To check what a label resolves to, refer to the [nextflow-commons.config](nextflow.config). 

### Examples of label resolution

#### label `'cpu_8'`: 

```bash
    withLabel:cpu_8 {
        cpus   = { check_max( 8, 'cpus' ) }
    }
```
- This resolves to 8 cpus.

#### label `'mem_32'`:

```bash
    withLabel:mem_32 {
        memory = { check_max( escalate_exp( 32.GB, task, 2 ), 'memory' ) }
    }
```
- Resolves to `32 GB memory`.
- Enables retries, escalating exponentially: `32 GB` → `64 GB` → `128 GB`.

#### label `'time_30m'`:

```bash
    withLabel:time_30m {
        time   = { check_max( escalate_linear( 30.m, task ), 'time' ) }
    }
```
- Resolves to `30 minutes`.
- Enables retries with linear escalation: `30 m` → `60 m` → `90 m`, etc.

#### label `'time_queue_from_normal'`:

```bash
    withLabel:time_queue_from_normal {
        time = { check_max( escalate_queue_time( 'normal', task ), 'time' ) }
    }
```
- Resolves to `12 hours` maximum task job runtime initially.
- If the task exceeds the maximum runtime, the `time` and `queue` task parameter values are updated during retries:
`12.h` time and `normal` queue → `48.h` time and `long` queue → `7.d` time and `week` queue → `30.d` and `basement` queue

Note: Even if you manually supply a `queue` parameter value, it will be overridden by this label when the task is retried, so please be careful. You should thus always provide an overriding value of the `queue` parameter together with the `time` parameter. You may also - but it's not recommended - redefine the effect of the `time_queue_from_normal` label in your add-on config file to avoid this conflict arising for the target process, but beware of spillover effects on other processes that would share this label.


**Tips:** 

* Verify label definitions to understand resource allocations and retry behavior as labels can override manually specified parameters.
