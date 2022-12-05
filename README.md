## TL;DR
The objective of this document is to provide a detailed design of the solution for GPU cluster provisioning to run AI/ML workloads. This solution is designed for A2+/A3 timeline. In the following sections we will compare different available options and then dive deep into the proposed solution. The instructions can be found [here](#instruction).
## Customer Usage Journey
Cluster provisioning effort aims to provide a solution for the external users to provision a GPU cluster quickly and efficiently and run their AI/ML workload in minutes. Similarly it aims to provide an automated way of providing GPU clusters for the internal AI/ML pipeline.

## Detailed Design
The AI Accelerator experience team provides docker images which provision the cluster when run as a container. The docker image is self contained to create all the necessary resources for a GPU cluster. It has all the configs prepackaged and the tools installed. The configs packaged inside the image define the baseline GPU cluster. 

### Baseline cluster configuration
The baseline GPU cluster is the collection of resources recommended/supported by the AI accelerator experience team. Examples of that include Supported VM types, accelerator types, VM images, shared storage solutions like GCSFuse etc. These are first tested within the AI Accelerator experience team and then they are integrated with the cluster provisioning tool. The way they are incorporated into the tool is via Terraform configs packaged within the docker container. In some cases these features can be optional and users may choose to use it (eg: GCSFuse) but in some other cases they will be mandated by AI Accelerator exp. team (eg: DLVM image).  

The default GPU cluster that gets created by the cluster provisioning tool is a single instance VM of type “a2-highgpu-2g” with 2 “Nvidia-tesla-a100” GPUs attached to it. It uses pytorch-1-12-gpu-debian-10 image. There is no startup script or any orchestrator (like Ray) set up. The jupyter notebook endpoint is accessible for this VM instance. There is a bucket created in the project provided by the user to manage the terraform state.  Users can create more advanced clusters using configuration described below.  

### Configuration for Users
Users have control to choose values for different fields for the resources. The mandated parameters are 
1. PROJECT_ID: The project ID to use for resource creation. 
2. NAME_PREFIX: The name prefix to use for creating the resources. This is the unique ID for the clusters created using the provisioning tool. 
3. ZONE: The zone to use for resource creation.

The optional parameters are
1. ***INSTANCE_COUNT***. This defines the VM instance count. The default value is 1 if not set.
2. ***GPU_COUNT***. This defines the GPU count per VM. The default value is 2 if not set.
3. ***VM_TYPE***. This defines the VM type. The default value is a2-highgpu-2g if not set.
4. ***ACCELERATOR_TYPE***. This defines the Accelerator type. The default value is nvidia-tesla-a100 if not set.
5. ***IMAGE_FAMILY_NAME***. This defines the image family name for the VM. The default value is pytorch-1-12-gpu-debian-10 if not set. We support images only from ml-images project.
6. ***IMAGE_NAME***. This defines the image name for the VM. The default value is c2-deeplearning-pytorch-1-12-cu113-v20221107-debian-10 if not set. We support images only from ml-images project.
7. ***DISK_SIZE_GB***. This defines the disk size in GB for the VMs. The default value is 2000 GB(2 TB) if not specified.
8. ***DISK_TYPE***. This defines the disk type to use for VM creation. The default value is pd-ssd if not defined.
9. ***GCS_PATH***. Google cloud storage bucket path to use for state management and copying scripts. If not provided then a default GCS bucket is created in the project. The name of the bucket is ‘aiinfra-terraform-<PROJECT_ID>’. For each deployment a separate folder is created under this GCS bucket in the name ‘<NAME_PREFIX-deployment>’. Ex: gs://spani-tst/deployment
10. ***COPY_DIR_PATH***. This defines the destination directory path in the VM for file copy. If any local directory is mounted at "/usr/aiinfra/copy" in the docker container then all the files in that directory are copied to the COPY_DIR_PATH in the VM. If not specified the default value is '/usr/aiinfra/copy'.
11. ***METADATA***. This defines optional metadata to be set for the VM. Ex: { key1 = "val", key2 = "val2"}
12. ***LABELS***. This defines key value pairs to set as labels when the VMs are created. Ex: { key1 = "val", key2 = "val2"} 
13. ***STARTUP_COMMAND***. This defines the startup command to run when the VM starts up. Ex: python /usr/cp/train.py
14. ***STARTUP_SCRIPT_PATH***. This defines the path of the startup script in the container. The script gets executed when the VM starts.Local directory can be mounted into the container and the startup script path can be provided accordingly. Ex: /usr/cp/test.sh
15. ***ORCHESTRATOR_TYPE***. This defines the Orchestrator type to be set up on the VMs. The current supported orchestrator type is: Ray.
16. ***SHOW_PROXY_URL***. This controls if the Jupyter notebook proxy url is retrieved for the cluster or not. If this is present and set to no, then connection information is not collected. The supported values are: yes, no.

The user needs to provide value for the above mandatory parameters. All other parameters are optional and default behaviour is described above. Users can also enable/disable various features using feature flags in the config, for example: ORCHESTRATOR_TYPE, SHOW_PROXY_URL, GCSFuse, Multi-NIC VM etc. The configuration file contains configs as key value pairs and provided to the ‘docker run’ command. These are set as environment variables within the docker container and then entrypoint.sh script uses these environment variables to configure terraform to create resources accordingly. 

#### Sample config file that the user provides.
``` 
# -----------------------------------------------------------------------------------------------
# Required environment variables.

# PROJECT_ID. This is the project id to be used for creating resources. Ex: supercomputer-testing
# NAME_PREFIX. This is the name prefix to be used for naming the resources. Ex: spani
# ZONE. This is the Zone for creating the resources. Ex: us-central1-f

# Optional Environment variables.

# INSTANCE_COUNT. This defines the VM instance count. The default value is 1 if not set.
# GPU_COUNT. This defines the GPU count per VM. The default value is 2 if not set.
# VM_TYPE. This defines the VM type. The default value is a2-highgpu-2g if not set.
# ACCELERATOR_TYPE. This defines the Accelerator type. The default value is nvidia-tesla-a100 if not set.
# IMAGE_FAMILY_NAME. This defines the image family name for the VM. The default value is pytorch-1-12-gpu-debian-10 if not set. We support images only from ml-images project.
# IMAGE_NAME. This defines the image name for the VM. The default value is c2-deeplearning-pytorch-1-11-cu113-v20220701-debian-10 if not set. We support images only from ml-images project.
# DISK_SIZE_GB. This defines the disk size in GB for the VMs. The default value is 2000 GB(2 TB) if not specified.
# DISK_TYPE. This defines the disk type to use for VM creation. The default value is pd-ssd if not defined.
# GCS_PATH. Google cloud storage bucket path to use for state management and copying scripts. Ex: gs://spani-tst/deployment
# COPY_DIR_PATH. This defines the destination directory path in the VM for file copy. If any local directory is mounted at "/usr/aiinfra/copy" in the docker container then all the files in that directory are copied to the COPY_DIR_PATH in the VM. If not specified the default value is '/usr/aiinfra/copy'. 
# METADATA. This defines optional metadata to be set for the VM. Ex: { key1 = "val", key2 = "val2"}
# LABELS. This defines key value pairs to set as labels when the VMs are created. Ex: { key1 = "val", key2 = "val2"} 
# STARTUP_COMMAND. This defines the startup command to run when the VM starts up. Ex: python /usr/cp/train.py
# STARTUP_SCRIPT_PATH. This defines the path of the startup script in the container. The script gets executed when the VM starts.Local directory can be mounted into the container and the startup script path can be provided accordingly. Ex: /usr/cp/test.sh
# ORCHESTRATOR_TYPE. This defines the Orchestrator type to setup on the VMs. The current supported orchestrator type is: Ray.
# SHOW_PROXY_URL. This controls if Jupyter notebook proxy url is retrieved for the cluster or not. If this is present and set to no, then connection information is not collected.The supported values are: yes, no.
# -----------------------------------------------------------------------------------------------

ACTION=Create
PROJECT_ID=supercomputer-testing
NAME_PREFIX=spani
ZONE=us-central1-f
GCS_PATH=gs://spani-tst
COPY_DIR_PATH=/usr/cp
STARTUP_COMMAND=python /tmp/test.py
METADATA={ meta1 = "val", meta2 = "val2" }     # export METADATA="{ meta1 = \"val\", meta2 = \"val2\" }" if variable is not getting set via docker.
LABELS={ label1 = "marker1" }                  # export LABELS="{ label1 = \"marker1\" }" if variable is not getting set via docker.
ORCHESTRATOR_TYPE=Ray
INSTANCE_COUNT=3
```

### Setting up Terraform to create resources
The user updates the config file and runs the docker image with the config file to create resources using the ‘docker run’ command. As part of the run command, users have to specify an action. The action can be Create, Destroy, Validate or Debug. The sample command looks like
```
docker run -it --env-file env.list us-central1-docker.pkg.dev/supercomputer-testing/cluster-provision-repo/cluster-provision-image:latest Create

docker run -it --env-file env.list us-central1-docker.pkg.dev/supercomputer-testing/cluster-provision-repo/cluster-provision-image:latest Destroy
```
All the setup needed before calling terraform to create resources is handled by entrypoint.sh script. This is packaged in the docker image and gets executed when the container starts. The entrypoint script validates environment variables and errors out if required ones are not provided. After that it uses the environment variables values to create the ‘tf.auto.tfvar’ file which is used by terraform to create the resources.

### User Authentication
The cluster provisioning tool interacts with GCP to create cloud resources on behalf of the user. So for doing that it needs the user’s authentication token with GCP. There are 3 environments where we expect the cluster provisioning tool to run. They are

1. Cloud shell: In the cloud shell environment, the default cloud authentication token is available that cluster provisioning tool uses for resource creation. No additional action is needed from the user. 
2. User machine with gcloud authentication: In the case where the user is running the cluster provisioning tool on their machine, they need to be authenticated with GCP for resource creation. There are 2 ways to do this.
   - Running ‘gcloud auth application-default login’ and mounting the local gcloud config to the container. For that use the  ‘docker run’ command with option ‘-v ~/.config/gcloud:/root/.config/gcloud ’.
   - Simply run the container image using ‘docker run’. When the cluster provisioning tool does not find any default authentication token it asks for manual authentication. Follow the prompt and provide the authorization code. The authorization prompt looks like below which is the same as gcloud authorization prompt.
    ```
    ================SETTING UP ENVIRONMENT FOR TERRAFORM================
    Setting Action to destroy
    Found Project ID soumyapani-testing
    ERROR: (gcloud.projects.describe) You do not currently have an active account selected.
    Please run:
    
      $ gcloud auth login
    
    to obtain new credentials.
    
    If you have already logged in with a different account:
    
        $ gcloud config set account ACCOUNT
    
    to select an already authenticated account to use.
    Failed to get project number. Return value = 1
    No authenticated account found.
    Go to the following link in your browser:
    
        https://accounts.google.com/o/oauth2/auth?...........
    
    Enter authorization code: 4/0Afxxxxxxxxxxxx
    ```
3. LLM pipeline: LLM pipeline uses VMs to run the cluster provisioning tool and the VM has default cloud authentication token available. So additional authentication is not needed here.


### State management across sessions
When terraform is executed to create the resources, it creates state files to keep track of the created resources. While changing or destroying resources, terraform uses these statefiles. If these state files are created within the container, then they will be lost when the container exits. That will result in leaking of the resources. Having the state files in the container forces cleanup resources before the container exits. This ties the cluster lifespan to that of the container. So to have better control over the resources we need to have the state files managed in cloud storage. That way multiple runs of the container can use the same state files to manage the resources. 
Using the provisioning tool the user can provide a GCS bucket path that the entrypoint.sh script uses for managing the terraform state and sets it as the backend for terraform. If the user does not provide a GCS bucket path then the provisioning tool creates a GCS bucket for managing terraform state, but for this the user needs to have permission to create a GCS bucket in the project they are using.

### Copying AI/ML training script to the GPU cluster
There are 2 supported ways to copy training scripts to the GPU cluster. 
1. The first way is via copying scripts from the local directory. For that
   - First the user needs to mount a local directory containing training scripts to ‘/usr/aiinfra/copy ’ location. To do that use the ‘docker run’ command with option ‘-v /localdirpath:/usr/aiinfra/copy ’
   - Then the user needs to provide the destination location as ‘COPY_DIR_PATH’ parameter. All the files under the mounted local directory will be copied to all the VMs under the path provided. If COPY_DIR_PATH’  is not provided then the default destination path is ‘/usr/aiinfra/copy ’ in the VM. 
2. The second method is via GCSFuse. Users can simply provide their GCS bucket where they can have training scripts and data via the ‘GCS_PATH’ parameter. Cluster provisioning tool will mount the GCS bucket in the VM as a local volume using GCSFuse. This feature is under development and details will be provided once available. 
#### Multi-node training
For multi-node training, we need to set up an orchestrator on all the VMs of the GPU cluster. Users can choose the orchestrator via ‘ORCHESTRATOR_TYPE’ parameter. Currently we support only Ray as our orchestrator. We will be adding support for more orchestrator types like Slurm shortly.

### Connecting to the GPU cluster and running the training script
Jupyter notebook is the default and recommended way to connect to the GPU cluster. All the VMs that get created through the cluster provisioning tool have proxy enabled for jupyter notebook. As part of the DLVM image, jupyter notebook server is started when the VM is created and a proxy url is created to access the notebook endpoint. After successfully creating the VMs, the cluster provisioning tool waits for the jupyter notebook server to be up and provides the url to connect, which looks like below. 
```
Apply complete! Resources: 7 added, 0 changed, 0 destroyed.
Terraform apply finished successfully.
Jupyter notebook endpoint not available yet. Sleeping 15 seconds.
Jupyter notebook endpoint not available yet. Sleeping 15 seconds.
 Terraform state file location: 
gs://aiinfra-terraform-soumyapani-testing/spani4-deployment/terraform/state
 Use below links to connect to the VMs: 
spani4-vm-gh9l:https://1896669fce99a2c1-dot-us-central1.notebooks.googleusercontent.com
spani4-vm-nrcv:https://11a0dd452fdf76d3-dot-us-central1.notebooks.googleusercontent.com
```
The user can use this URL on their browser to connect to the jupyter notebook and execute their training script.
There are some default training scripts provided in the VMs under location ‘/usr/aiinfra/sample’. Users can run those scripts after connecting to the VM to see them in action. This feature is under development and details will be provided once available. 

### Resource cleanup
Since the resource state is stored outside of the container, the GPU cluster lifespan is decoupled from the container’s lifespan. Now the user can run the container and provide ‘ACTION=Create’ as part of the ‘docker run’ command to create the resources. They can run the container again and provide ‘ACTION=Destroy’ to destroy the container. The terraform state stored in the GCS bucket is cleared when the destroy operation is called.

## Instruction
1. gcloud auth application-default login.
2. gcloud auth configure-docker us-central1-docker.pkg.dev
3. ***[`OPTIONAL - if project not set already`]*** gcloud config set account supercomputer-testing
4. Create env.list file. The sample env.list can be found [here](#sample-config-file-that-the-user-provides). 
5. ***[`SIMPLE CREATE`]*** 
   > docker run -it --env-file env.list us-central1-docker.pkg.dev/supercomputer-testing/cluster-provision-repo/cluster-provision-image:latest Create
6. ***[`SIMPLE DESTROY`]*** 
   > docker run -it --env-file env.list us-central1-docker.pkg.dev/supercomputer-testing/cluster-provision-repo/cluster-provision-image:latest Destroy
7. ***[`OPTIONAL - Pull docker image before hand`]*** 
   > docker pull us-central1-docker.pkg.dev/supercomputer-testing/cluster-provision-repo/cluster-provision-image:latest
8. ***[`OPTIONAL - Mount local directory`]*** 
   > docker run -v /usr/soumyapani/test:/usr/aiinfra/copy -it --env-file env.list us-central1-docker.pkg.dev/supercomputer-testing/cluster-provision-repo/cluster-provision-image:latest Create
9.  ***[`OPTIONAL - Mount gcloud config for auth token`]*** 
    > docker run -v ~/.config/gcloud:/root/.config/gcloud -it --env-file env.list us-central1-docker.pkg.dev/supercomputer-testing/cluster-provision-repo/cluster-provision-image:latest Create
10. ***[`OPTIONAL - GCS bucket not provided`]*** 
    > Need storage object owner access if you don't already have a storage bucket to reuse.

## Known Issues
1. Error: Error waiting for Deleting Network: The network resource 'projects/xxx' is already being used by 'projects/firewall-yyy’
This is due to a known bug in VPC b/186792016.
