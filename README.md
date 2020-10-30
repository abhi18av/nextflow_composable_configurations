# Composable configurations


The `nextflow.config` file plays a major role in production infrastructure while giving us tools such as `profiles` to keep the workloads portable.

This proposal outlines an approach towards constructing the **configuration** files with **composability** and **modularity** in mind.


## Summary

To avoid long and messy configuration files and honor the `DRY` principle, we can simplify the configuration management using a concept similar to *nextflow modules*.

A possible solution could be a combination of  the top-level `nextflow.config` and `configs` folder shown below in the directory tree, which can allow us to keep the overall configuration consistent and composable.


```

├── configs
│   ├── profiles
│   │   ├── dockerhub.config
│   │   ├── gcp_dev.config
│   │   └── gcp_prod.config
│   └── scopes
│       ├── docker.standard.config
│       └── process.standard.config
├── nextflow.config
└── params.yaml




```



## TIP

To develop and debug the `configuration`, checkout the `-c` (soft-override), `-C` (hard-override) top-level options and `config`command on the Nextflow CLI documentation.

You might find yourself using the following command quite often while working on the configuration

``` 
$ nextflow config -a

```




## Problem-1

**How to avoid long configuration files?**

Once we start thinking about portability and optimizations, top-level `nextflow.config` has a tendency to grow too much. The situation gets complicated once we go beyond the local development environment and have to test and optimize the pipeline across multiple clouds/executors - `profiles` have become a standard way to achieve this.


Take the following as an example 

```nextflow

manifest {
    name = 'Analysis'
    version = '0.0.1'
    author = 'some_author'
    mainScript = 'analysis.nf'
    defaultBranch = 'master'
    nextflowVersion = '>=20.01.0-edge'
}

profiles {
  standard {
    process {
      errorStrategy = 'finish'
    }
  }
  dockerhub {
    docker.enabled = true
    docker.fixOwnership = true
    process {
      errorStrategy = 'retry'
      maxRetries = 3
      ext.registry = 'dockerhub.com'
      ext.project = 'local'
    }
  }
  gcp_prod {
    google.project = 'project1'
    google.zone = 'europe-west2-c'
    cloud.preemptible = true
    executor.queueSize = 48
    workDir = 'gs://nextflow-work-bucket/work'
    process {
      executor = 'google-lifesciences'
      errorStrategy = { task.exitStatus==14 ? 'retry' : 'terminate' }
      maxRetries = 3
      ext.registry = 'gcr.io'
      ext.project = 'project1'
      ext.results = "$baseDir/results/prod"
    }
  }
  gcp_dev {
    tower {
      accessToken = 'i_dont_work_d26b1c89d4c0ac1a958c005f289083d6'
      enabled = true
    }
    google {
      zone = 'europe-west2-c'
      lifeSciences.debug = false
      lifeSciences.preemptible = false
      lifeSciences.usePrivateAddress = true
    }
    executor.queueSize = 192
    bucketDir = 'gs://nextflow-work-bucket/work'
    process {
      executor = 'google-lifesciences'
      errorStrategy = 'retry'
      maxRetries = 5
      ext.registry = 'gcr.io'
      ext.project = 'project1-dev'
      ext.results = "$baseDir/results/dev"
    }
  }
}


```

## Solution-1

Use the `includeConfig` feature to tone down the above file to something like the following 


```nextflow

manifest {
    name = 'Analysis'
    version = '0.0.1'
    author = 'some_author'
    mainScript = 'analysis.nf'
    defaultBranch = 'master'
    nextflowVersion = '>=20.01.0-edge'
}

profiles {
  standard {
    process {
       includeConfig "./config/scopes/process.standard.config"
    }
  }
  
    // Profile for dockerhub
    includeConfig "./config/profiles/dockerhub.config"


    // Profile for gcp_dev
    includeConfig "./config/profiles/gcp_dev.config"


    // Profile for gcp_prod
    includeConfig "./config/profiles/gcp_prod.config"


}


```

This allows us to make use of the `includConfig` and the automatic **configuration resolution** which *Nextflow* offers us by default. 

We can achieve this composability by relying on certain conventions, for example in the above example, we have relied on a separate folder called `configs` as a base for the **profiles** and **scopes**, as defined in the official Nextflow documentation. 

The proposal is to have a folder structure similar to the following and treating the `standard` profile as the ground truth for all other profiles (the reason that scopes files have a `.standard.` in their names).

``` 

configs
├── profiles
│   └── dockerhub.config
└── scopes
    ├── docker.standard.config
    └── process.standard.config


```




## Problem-2

**How do we keep the configuration `DRY`?**

## Solution-2

Note that there are `scope-fields` which are similar in multiple `profiles`, such as `process.errorStrategy` in the case of `standard`, `dockerhub`, `gcp_prod` and `gcp_dev` though the value of `process.errorStrategy` field (sometimes) differ in these profiles.

It is worth noting that since we are treating the `standard` profile as the ground truth, we rely upon the `soft-override` of properties in the derived profiles.


For example, our `./configs/scopes/process.standard.config` could be something like 


``` nextflow
errorStrategy = 'finish'

withName:
"GATK_*" {
    memory = 8
    cpus = 8
}


```

And we can include this within the `standard` profile in the top-level `nextflow.config` 

``` nextflow
standard {
    process {
       includeConfig "./config/scopes/process.standard.config"
    }
}

```


To override this default, we can follow the same pattern in the `./configs/profiles/dockerhub.config` file

``` nextflow
dockerhub {
    process {
        // Include the standard process 
        includeConfig "../scopes/process.standard.config"
        
        // Override the standard errorStrtegy
        errorStrategy = 'retry
        
        // Add new field specific to this profile
        maxRetries = 3 


        // Append new field specific to this profile
        withName:
        "GATK_*" {
            container = "broadinstitute/gatk:4.1.8.1"
        }

    }

}


```


In the above snippet, we override the `errorStrategy` field from the `process.standard.config` and then add the `maxRetries` field. Moreover, since these are `soft-overrides` whatever value we  don't override in the `dockerhub` scope will be the same as `standard` profile, which means that the `cpus` and `memory` for `GATK_*`  processes is still 8, we only specified that within the `dockerhub` profile we'd like to run these processes with the `broadinstitute/gatk:4.1.8.1` container.



## TIP

At this point you might be wondering **should we always derive any new profile directly from `standard` scopes?**

The answer to this is, **mostly yes**, though it also depends on how you wish to design it and what's manageable for you and your team. You could also do something like 

``` 
├── configs
│   ├── profiles
│   │   ├── dockerhub.config
│   │   └── gcp
│   │       ├── gcp_dev.config
│   │       ├── gcp_prod.config
│   │       └── utils.config

```

...but it ends up increasing the nesting too much. As a heuristic, it's recommended that you compose the profiles using the standard scopes and then override/add the fields which are specific to that pipeline.
