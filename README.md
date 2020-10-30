# Composable configurations


The `nextflow.config` file plays a major role in production infrastructure while giving us tools such as `profiles` to keep the workloads portable.

This proposal outlines an approach towards constructing the **configuration** files with **composability** and **modularity** in mind.


## Summary

To avoid long and messy configuration files and honor the `DRY` principle, we can simplify the configuration management using a concept similar to *nextflow modules*.

A possible solution could be a combination of  the top-level `nextflow.config` and `configs` folder shown below in the directory tree, which can allow us to keep the overall configuration consistent and composable.


```
params.yaml

nextflow.config

configs
├── profiles
│   └── dockerhub.config
└── scopes
    ├── docker.standard.config
    └── process.standard.config


```


## Problem-1

Once we start thinking about portability and optimizations, top-level `nextflow.config` has a tendency to grow too much.


```nextflow

manifest {
    name = 'Analysis'
    version = '0.0.1'
    author = 'some_author'
    mainScript = 'analysis.nf'
    defaultBranch = 'master'
    description = 'Primary Analysis Pipeline for Cambridge Epigenetix'
    homePage = 'https://cambridge-epigenetix.com/'
    nextflowVersion = '20.01.0-edge'
}

profiles {
  standard {
    process {
      errorStrategy = 'finish'
    }
  }
  docker {
    docker.enabled = true
    docker.fixOwnership = true
    process {
      ext.registry = 'gcr.io'
      ext.project = 'cegx-pilot-260309'
    }
  }
  gcp_cegx {
    google.project = 'cegx-test-project1'
    google.zone = 'europe-west2-c'
    cloud.preemptible = true
    executor.queueSize = 48
    workDir = 'gs://nextflow-cegx-work-bucket/work'
    process {
      executor = 'google-lifesciences'
      errorStrategy = { task.exitStatus==14 ? 'retry' : 'terminate' }
      maxRetries = 3
      ext.registry = 'gcr.io'
      ext.project = 'cegx-test-project1'
       ext.results = "$baseDir/results/CEG752011"
    }
    params {
      base_path="gs://nextflow-pilot/test_data"
      reads = "$params.base_path/reads/CEG*-*-*_S*_L*_R{1,2}_*.fastq.gz"
      bwa_index = "params.base_path/bwa_index/GRCh38.no_alt_analysis_set.HMCP.v10.chr21.fa.bwa-index.tar.gz"
      karyo_features = "params.base_path/features/GRCh38.karyo.bed"
      spikein_features = "params.base_path/features/HMCP.20180620.native.bed"
      hmcp_ref = "params.base_path/hmcp/HMCP.20180620.native.fa"
      hmcp_ref_idx ="params.base_path/hmcp/HMCP.20180620.native.fa.fai"
    }
  }
  gcp_seqera {
    tower {
      accessToken = 'fba5bad7d26b1c89d4c0ac1a958c005f289083d6'
      enabled = true
    }
    google {
      zone = 'europe-west2-c'
      lifeSciences.debug = false
      lifeSciences.preemptible = false
      lifeSciences.usePrivateAddress = true
    }
    executor.queueSize = 192
    bucketDir = 'gs://nextflow-cegx-work-bucket/work'
    process {
      executor = 'google-lifesciences'
      errorStrategy = 'retry'
      maxRetries = 5
      ext.registry = 'gcr.io'
      ext.project = 'cegx-pilot-260309'
      ext.results = "$baseDir/results/CEG752011"
    }
    params {
      base_path="gs://nextflow-cegx-work-bucket/test_data"
      reads = "$params.base_path/reads/CEG*-*-*_S*_L*_R{1,2}_*.fastq.gz"
      bwa_index = "$params.base_path/bwa_index/GRCh38.no_alt_analysis_set.HMCP.v10.chr21.fa.bwa-index.tar.gz"
      karyo_features = "$params.base_path/features/GRCh38.karyo.bed"
      spikein_features = "$params.base_path/features/HMCP.20180620.native.bed"
      hmcp_ref = "$params.base_path/hmcp/HMCP.20180620.native.fa"
      hmcp_ref_idx ="$params.base_path/hmcp/HMCP.20180620.native.fa.fai"
    }
  }
}


```

