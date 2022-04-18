# Running the WASHU Immunogenomics Workflow on Google Cloud - public version

## Preamble
This tutorial demonstrates how to run the WASHU immunogenomics pipeline (immuno.wdl) on Google Cloud.

The same principles described in this tutorial should work for any of our collection of WDL files found here: https://github.com/griffithlab/analysis-wdls.

This workflow run will be accomplished by first setting up workflow definitions, input data and 
reference files and a YAML config file on a user's local system. The user will also set up a Google Cloud 
environment and all the inputs will be staged to this cloud environment. Next Cromwell will be used to execute the pipeline using the specified input and reference files, and finally the results will be pulled back to the local system and cloud resources will be cleaned up. 

Some steps are run within docker containers. It therefore requires that you have the ability to run interative docker sessions on your system.

You will interact with the Google Cloud in several ways:
1. From your local system using command line tools, including `gcloud` and `gsutil`.
2. We will create a Google Virtual Machine (VM) and start a Cromwell service on it. You will then login to this VM and perform some commands and monitoring of workflow progress there. Cromwell on this VM will orchestrate creation and use of many additional worker VMs that complete all the compute tasks of the workflow.
3. The Google Cloud Console in your web browser may be used to visualize/monitor usage of cloud resources. In the Console you will see most relevant resources in the "Cloud Storage" and "Compute Engine" sections.

This version assumes that your are staging your input data files from your local system (e.g. a laptop, but could be any system really). All input files will be staged to a Google Storage Bucket. During the workflow, all results files will be also be stored in this Bucket. At the end of the tutorial the final results will be copied back to your local system.

After completing the workflow, ALL resources used on the cloud can be destroyed. One possible exception to this may occur if you need custom reference files that you wish to persist for many analyses. However, for this tutorial reference files will be accessed from a separate public bucket that we have created to support this workflow.

A brief note on command line sessions. Almost everything below will occur at the command line. It is very easy to get confused about the kind of sessions used. There are three types that will be used:
1. A terminal session on your local system (e.g. using the Terminal App on a Mac laptop)
2. Within session (1) you may launch a docker interative session (via `docker run -it`) to use tools we have created for setting up the workflow
3. Within session (1) you may login (via `gcloud compute ssh`) to the Google Virtual Machine where Cromwell is running.

### Source of instructions
This tutorial is a specific example of how to run a specific pipeline (immuno) on a specific example dataset (HCC1395 Tumor/normal cell line pair). The steps below are taken from the following link where you will find a more generic set of documentation that explains in detail how to run any WDL pipeline on the Google Cloud using tools created to assist this process. 
https://github.com/griffithlab/cloud-workflows/tree/main/manual-workflows

### Prerequisites
- docker
- google-cloud-sdk
- git

### Example data set and analysis to be performed
To demonstrate an analysis on the Google cloud we will run the WASHU immunogenomics pipeline on a publicly available set of exome and bulk RNA-seq data generated for a tumor/normal cell line pair (HCC1395 and HCC1395/BL). The HCC1395 cell line is a well known breast cancer cell line that can be purchased and is commonly used for benchmarking cancer genomics analysis and methods development. The datasets we will use here are realistic deeply sequenced exome and RNA-seq data. The immunogenomics pipeline is a very elaborate end-to-end pipeline that starts with raw data and performs data QC, germline variant calling, somatic variant calling (multiple variant callers and variant types), HLA typing, RNA expression analysis and neoantigen identification.  

### Setting up a Google Cloud account
In order to do analysis on the Google Cloud, you will need an account. Ideally this will be an account linked to your institution/company. Before proceeding, you should set up this account, log into it and make sure you have access to billing information: https://console.cloud.google.com/.

Some notes on account set up once your are logged in:
- Create a new project. In this tutorial we use project name: 'test-immuno'. When using the Google Web Console, remember to select this project in the top left corner.
- Create billing alerts! In the Google Cloud Web Console, select: Billing -> Budgets & alerts -> Create Budget. How you set up your alerts will depend on your anticipated level of use/expenditure. For example, you might set at $500 budget and then set up alerts to be sent at 50%, 100%, 200%, ..., X% of that budget.
- Choose a name for the Google bucket that will be used for this tutorial. Note that you don't need to create it in the console because this will be handled automatically for you below. We will use the bucket name 'test-immuno-pipeline' below.
 
Some notes on quotas:
- If you have not been using your account for high performance computing, it is likely that by default you have quotas in place that will cause the very large immuno.wdl workflow to fail. For example, you may not be able to use enough CPUs, IP addresses and disk space at once to get this workflow to run. If you see errors that mention quotas, or sound like fundamental network failures, check your quotas in the Google Cloud Console (IAM & Admin -> Quotas) and work with your Google account contact to increase your quotas. If your institution has already been using Google Cloud in any serious way, this is less likely to be a problem. 

Example quotas you might need to request:
- `cpus` -> `us-central1` -> `100`
- `preemptible_cpus` -> `us-central1` -> `100`
- `In-use IP addresses` -> `us-central1` -> `25`
- `Persistent Disk SSD (GB)` -> `us-central1` -> `10 TB`

### Interacting with Google buckets from your local system
Note that, if needed, you can use the following docker image to access `gsutil` for exploration of your google storage: `docker(google/cloud-sdk)`. Or alternatively, you can install the Google Cloud SDK on your system. This latter approach is assumed by the following instructions.

## Step-by-step instructions
Start by opening a Terminal session on your local system

### Set some Google Cloud and other environment variables
The following environment variables are used merely for convenience and should be customized to produce intuitive labeling for your own analysis:

```bash
export GCS_PROJECT=test-immuno
export GCS_SERVICE_ACCOUNT=cromwell-server@test-immuno.iam.gserviceaccount.com
export GCS_BUCKET_NAME=test-immuno-pipeline
export GCS_BUCKET_PATH=gs://test-immuno-pipeline
export GCS_INSTANCE_NAME=mg-immuno-test
export WORKING_BASE=~/Desktop/pipeline_test/gcp_wdl_test
export RAW_DATA_DIR=~/Desktop/pipeline_test/raw_data
export WORKFLOW_DEFINITION=$WORKING_BASE/git/analysis-wdls/definitions/immuno.wdl
export LOCAL_YAML=hcc1395_immuno_local-WDL.yaml
export CLOUD_YAML=hcc1395_immuno_cloud-WDL.yaml
```

Note that $GCS_SERVICE_ACCOUNT should contain the value in $GCS_PROJECT as in the example above. Also note that Google Bucket names must be **globally** unique. So, you will want to change $GCS_BUCKET_NAME and $GCS_BUCKET_PATH above to something specific to you. 

## Local setup

### First create a working directory on your local system
The following directory on the local system will contain: (a) a git repository for this tutorial including an example YAML file set up to work with the test HCC139 data, (b) git repository for the WDL workflows, including the immuno worflow, (c) git repository for tools that help provision and manage our workflow runs on the cloud, (d) raw data that we will download for this tutorial, (e) a YAML file describing the input data and paramters for the analysis, and (f) final results file from the workflow that we will pull down from the cloud after a successful run.
```bash
mkdir $WORKING_BASE
cd $WORKING_BASE
```

### Clone git repositories that have the workflows (pipelines) and scripts to help run them
The following repositories contain: this tutorial (immuno_gcp_wdl), the WDL workflows (analysis-wdls), and tools for running these on the cloud (cloud-workflows).
```bash
mkdir git
cd git
git clone git@github.com:griffithlab/immuno_gcp_wdl_local.git
git clone git@github.com:griffithlab/analysis-wdls.git
git clone git@github.com:griffithlab/cloud-workflows.git
```

### Login to GCP and set the desired project
From the command line, you will need to authenticate your cloud access (using your google cloud account credentials). This generally only needs to be done once, though there is no harm to re-authenticating. The login command below will generate a custom URL to enter in your browser. Once you do this, you will be prompted to log into your Google account. If you have multiple Google accounts (e.g. one for your institution/company and a personal one) be sure to use the correct one.  Once you have logged in you will be presented with a long code. Enter this at the prompt generated by the login command below. Finally, set your desired Google Project. This Project should correspond to a Google Billing account in the Google console. If you are using Google Cloud for the first time, both billing and a project should be set up before proceeding. Configuring billing alerts is also probably wise at this point.
```bash
gcloud auth login
gcloud config set project $GCS_PROJECT
gcloud config list
```

The `gcloud config list` command can be used to remind yourself how you are currently authenticated to use Google Cloud Services. This can be helpful because on your host machine, you will be authenticated using your personal account. However, on the Google VM where the workflow will be orchestrated by Cromwell, you will be authenticated using a "service" account. 

### Set up cloud service account, firewall settings, and storage bucket
Run the following command and make note of the "Service Account" returned (e.g. "cromwell-server@test-immuno.iam.gserviceaccount.com"). Make sure this matches the value in $GCS_SERVICE_ACCOUNT (e.g. `echo $GCS_SERVICE_ACCOUNT`).

```bash
cd $WORKING_BASE/git/cloud-workflows/manual-workflows/
bash resources.sh init-project --project $GCS_PROJECT --bucket $GCS_BUCKET_NAME
```

This step should have created two new configuration files in your current directory: `cromwell.conf` and `workflow_options.json`.

### Download input data files

Download RAW data for hcc1395
```bash
mkdir -p $RAW_DATA_DIR/hcc1395
cd $RAW_DATA_DIR/hcc1395
wget http://genomedata.org/pmbio-workshop/fastqs/all/Exome_Norm.tar
wget http://genomedata.org/pmbio-workshop/fastqs/all/Exome_Tumor.tar
wget http://genomedata.org/pmbio-workshop/fastqs/all/RNAseq_Tumor.tar
tar -xvf Exome_Norm.tar 
tar -xvf Exome_Tumor.tar 
tar -xvf RNAseq_Tumor.tar
rm -f Exome_Norm.tar Exome_Tumor.tar RNAseq_Tumor.tar
```

### Obtain an example configuration (YAML) file on local system
Create a directory for YAML files and create one for the desired pipeline that points to the location of input files on your local system

```bash
cd $WORKING_BASE
mkdir yamls
cd yamls
```

Setup yaml files for an example run.
```bash
cp $WORKING_BASE/git/immuno_gcp_wdl_local/example_yamls/human_GRCh38_ens105/hcc1395_immuno_local-WDL.yaml $WORKING_BASE/yamls/
```

Note that this YAML file has been set up to work with the HCC1395 raw data files downloaded above. You will need to update the PATHs to the FASTQ files to match the locations you downloaded them to above.

Open the YAML file with an editor and correct all 8 lines that contain paths to input data FASTQ files.

If you are modifying this tutorial to work with your own data, you will need to modify the YAML lines that relate to input sequence files.  For both DNA and RNA files, both FASTQ and Unaligned BAM files are supported as input.  Similarly, you have have your data in one file (or one file pair) or you may have multiple data files that will be merged together. Depending on how your input data is organized the YAML entries will look slightly different.

### Create a copy of reference and annotation files in your Google Bucket
In the following step, we will make a copy of a bundle of reference files from one Public ("Requestor Pays") Google Bucket into our own Google Bucket as follows:

```bash
gsutil -u $GCS_PROJECT ls gs://griffith-lab-workflow-inputs/
gsutil -u $GCS_PROJECT cp -r gs://griffith-lab-workflow-inputs/human_GRCh38_ens105 $GCS_BUCKET_PATH/human_GRCh38_ens105
gsutil ls $GCS_BUCKET_PATH/human_GRCh38_ens105

```

Note that in the commands above, we must use the `-u` option to specify a project for billing to access a public Google Bucket configured as "Requestor Pays".

### Stage input data files to cloud bucket

The reference and annotation files were already in a Google Bucket and we simply had to copy them from a public Bucket to our own. However, the input FASTQ data files are still on our local system, so we still need to upload them and update our YAML file to point to their new locations on the cloud.

Start an interactive docker session capable of running the "cloudize" scripts. Note that the following docker command uses `--env` commands to pass some convenient environment variables in that were exported above. The first `-v` option is used to allow access to where the data is stored. The second `-v` is used to expose the location of the Google Cloud Credential files.  You will need to update this to reflect your username. The `-it` option is used to make the session interactive and we specify to drop into a `/bin/bash` session. `mgibio/cloudize-workflow:latest` is the docker image we will use.

```bash
docker pull mgibio/cloudize-workflow:latest
docker run -it --env GCS_PROJECT --env GCS_BUCKET_NAME --env GCS_BUCKET_PATH --env GCS_INSTANCE_NAME --env GCS_SERVICE_ACCOUNT --env WORKING_BASE --env WORKFLOW_DEFINITION --env LOCAL_YAML --env CLOUD_YAML -v /Users/mgriffit/Desktop/pipeline_test/:/Users/mgriffit/Desktop/pipeline_test/ -v /Users/mgriffit/.config/gcloud:/root/.config/gcloud mgibio/cloudize-workflow:latest /bin/bash
```

Attempt to cloudize your workflow and inputs
```bash
cd $WORKING_BASE/yamls
python3 /opt/scripts/cloudize-workflow.py $GCS_BUCKET_NAME $WORKFLOW_DEFINITION $WORKING_BASE/yamls/$LOCAL_YAML --output=$WORKING_BASE/yamls/$CLOUD_YAML
```

Note that this "cloudize" step is primarily about staging input files that you may have locally on your system that need to be copied to a google cloud bucket so that they can be accessed during your workflow run on the cloud. It therefore takes the desired Google Cloud Bucket Name as input. It also takes your Workflow Definition (WDL file) for your pipeline.  It will perform some basic checks of the input expectations of this workflow against the input YAML configuration file you provide. The final input to this script is your input YAML configuration file (the LOCAL YAML). This should contain all the input parameters needed by your workflow, including paths to reference, annotation and data files. If any of the file paths in your input YAML are local paths, this script will attempt to copy them into your bucket. If a file path is already specified as a Google Cloud Bucket path (e.g. gs://PATH ) it will be skipped. The output of the script (the CLOUD YAML), is a version of your YAML that has been updated to point to new paths for any files that were copied into the cloud bucket. This is the YAML you will use to launch your workflow run in the following steps.

If you get an error during this step, a common cause is that there is some disconnect between what inputs you have defined in your YAML file and what the WDL workflow definition expects. Often the error message will hint at where to look in these two files.

### Start a Google VM that will run Cromwell and orchestrate completion of the workflow

Note that Cromwell produces a large quantity of database logging. To ensure we have enough space for a least a few runs and to localize intermediate and final results files from the workflow (which include numerous large BAMs) we will specify some extra disk space with `--boot-disk-size=250GB` (default would be 10GB). When not testing, this can probably be safely reduced to 20-40GB. Note that this argument must be listed last after the required arguments for the `start.sh` script.
```bash
cd $WORKING_BASE/git/cloud-workflows/manual-workflows/
bash start.sh $GCS_INSTANCE_NAME --server-account $GCS_SERVICE_ACCOUNT --project $GCS_PROJECT --boot-disk-size=250GB

#Now exit the interactive docker session
exit

```

### Log into the VM and check status 

In this step we will confirm that we can log into our Cromwell VM with `gcloud compute ssh` and make sure it is ready for use.

After logging in, use journalctl to see if the instance start up has completed, and cromwell launch has completed.

For details on how to recognize whether these processes have completed refer: [here](https://github.com/griffithlab/cloud-workflows/tree/main/manual-workflows#ssh-in-to-vm).

```bash
gcloud compute ssh $GCS_INSTANCE_NAME
journalctl -u google-startup-scripts -f
journalctl -u cromwell -f
exit
```

### Localize your inputs file

First **on your local system**, copy your cloudized YAML file to a google bucket

```bash
cd $WORKING_BASE/yamls/
gsutil cp $CLOUD_YAML $GCS_BUCKET_PATH/yamls/$CLOUD_YAML
```

Now log into Google Cromwell VM instance again and copy the YAML file to its local file system
```bash
gcloud compute ssh $GCS_INSTANCE_NAME

export GCS_BUCKET_PATH=gs://test-immuno-pipeline
export CLOUD_YAML=hcc1395_immuno_cloud-WDL.yaml

gsutil cp $GCS_BUCKET_PATH/yamls/$CLOUD_YAML .

``` 

### Run the immuno workflow using everything setup thus far

While logged into the Google Cromwell VM instance:
```bash
source /shared/helpers.sh
submit_workflow /shared/analysis-wdls/definitions/immuno.wdl $CLOUD_YAML

```

### Monitor progress of the workflow run:

While the job is running you can see Cromwell logs live as they occur by doing this
```bash
journalctl -f -u cromwell

```

### Save information about the workflow run itself - Timing Diagram and Outputs List

After a workflow is run, before exiting and deleting your VM, make sure that the timing diagram and the list of outputs are available so you can make use of the data outside of the cloud.

First determine you WORKFLOW_ID. This can be done several ways. If the run was successful it should be reported at the bottom of the cromwell log as "$WORKFLOW_ID  completed with status Succeeded". Or you find it by the name of the directory where your run was stored in the Google bucket. Both of these approaches are illustrated here:

```bash
export GCS_BUCKET_PATH=gs://test-immuno-pipeline
gsutil ls $GCS_BUCKET_PATH/cromwell-executions/immuno/

journalctl -u cromwell | tail | grep "Workflow actor"
```

Now save the workflow information in your google bucket
```bash
export WORKFLOW_ID=<id from above>
source /shared/helpers.sh
save_artifacts $WORKFLOW_ID $GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID
```

This command will upload the workflow's artifacts to your google bucket so they can be used after the VM is deleted. They can be found at paths:

```bash
$GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID/timing.html
$GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID/outputs.json
```

Confirm that they were successfully transferred and logout of the Cromwell VM on GCP:
```bash
gsutil ls $GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID
exit
```

The file `outputs.json` will simply be a map of output names to their GCS locations. The `pull_outputs.py` script can be used to retrieve the actual files.


### Pulling the Outputs from Google Cloud Bucket back to your local system or cluster
After the work in your compute instance is all done, including `save_artifacts`, and you want to bring your results back to the cluster, leverage the `pull_outputs.py` script with the generated `outputs.json` to retrieve the files.

On compute1 cluster, jump into a docker container with the script available

```bash
export WORKFLOW_ID=<id from above>

docker run -it --env WORKFLOW_ID --env GCS_BUCKET_PATH --env WORKING_BASE -v /Users/mgriffit/Desktop/pipeline_test/:/Users/mgriffit/Desktop/pipeline_test/ -v /Users/mgriffit/.config/gcloud:/root/.config/gcloud mgibio/cloudize-workflow:latest /bin/bash

```

Execute the script:

```bash
cd $WORKING_BASE
mkdir final_results
cd final_results

python3 /opt/scripts/pull_outputs.py --outputs-file=$GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID/outputs.json --outputs-dir=$WORKING_BASE/final_results/
exit
```

Examine the outputs briefly:
```bash 
cd $WORKING_BASE/final_results
ls -l
du -h

```

### Estimate the cost of executing your workflow

On compute1 cluster, start an interactive docker session as describe below and then use the following python script to generate a cost estimate:

```bash
export WORKFLOW_ID=<id from above>

docker run -it --env WORKFLOW_ID --env GCS_BUCKET_PATH --env WORKING_BASE -v /Users/mgriffit/Desktop/pipeline_test/:/Users/mgriffit/Desktop/pipeline_test/ -v /Users/mgriffit/.config/gcloud:/root/.config/gcloud mgibio/cloudize-workflow:latest /bin/bash

cd $WORKING_BASE/git/cloud-workflows/scripts
python3 estimate_billing.py $WORKFLOW_ID $GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID/metadata/
exit
```

### Once the workflow is done and results retrieved, destroy the Cromwell VM on GCP to avoid wasting resources

Use the following commmand to destroy the Cromwell VM. 

```bash
gcloud compute instances delete $GCS_INSTANCE_NAME
```

You can empty the cloud bucket either in the Web Console or using commands like `gsutil rm -r $GCS_BUCKET_PATH/folder-name`.

Finally, you should perform a survey of Cloud Storage and Compute Engine sections in the Google Cloud Web Console to make sure everything has been cleaned up successfully..

