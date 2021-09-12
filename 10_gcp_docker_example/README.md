# Launching On GCP with Jaynes and Docker

This folder contains a working example for launching jobs on the Google Cloud Platform (GCP) with docker containers. At the end of the day, you would have 1. a python script and 2. a simple `.jaynes` script that allows you to scale your experiment instantly to thousands of instances on the GCP.

**Example script:**

```python
import jaynes
from your_project import train, Args

for seed in [100, 200, 300]:
    jaynes.config(name=f"demo-instance/seed-{seed}")
    jaynes.run(train, seed=seed)
```

**Note**: The example config currently uses an S3 mount for the code upload. We currently do not have support for gce buckets, but that is an easy to implement. To add this support, submit a PR.



## Before You Begin

### Step 1: Installing `jaynes`

You need to have `gcloud` and `gsutil` installed on your computer, as well as `jaynes`. 

```bash
pip install jaynes
```

### Step 2: Installing the Cloud SDK (Google)

Then install and configure your `gcloud` and `gsutil` command line utilities according the these guides:

- install the cloud SDK (`gcloud`):  https://cloud.google.com/sdk/docs/install
- Install `gsutil`: https://cloud.google.com/storage/docs/gsutil_install
- [Set a default region and zone](https://cloud.google.com/compute/docs/gcloud-compute#set_default_zone_and_region_in_your_local_client).

Now after you have finished, you can verify that your cloud SDK is working via:

```bash
$ gcloud auth list
```

which should print out:

```
Credentialed Accounts
ACTIVE ACCOUNT
     * your-email@gmail.com
       your-other-email@gmail.com
```

To set the active account, run:
    

```
$ gcloud config set account <account>
```



## Machine Learning At Scale with `jaynes` on GCP

The following are supported in `jaynes>=v0.7.7` and above. See https://pypi.org/project/jaynes/0.7.7/

### Part 1: Creating A GCP Bucket for Your Code and Data

First make sure that you are able to run the `gsutil` command. Now, create two buckets using the following command:

```bash
gsutil mb gs://$USER-jaynes-$ORGANIZATION
gsutil mb gs://$USER-data-$ORGANIZATION
```

If you mess up, remember even if you delete a bucket, it would take a while for its name to be released, so that you can recreate it using different settings. Just don't panic!

```shell
gsutil rb gs://$USER-jaynes-$ORGANIZATION
gsutil rb gs://$USER-data-$ORGANIZATION
```

#### Using AWS S3 with GCE instances

The `aws cli` is not pre-installed on the machine learning GCE VM images. Therefore to download from AWS S3, you need to install the commandline tool as part of the `setup` step of your `.jaynes.runner` configuration. 

```yaml
launch: !ENVS
  setup: pip install -q awscli jaynes ml-logger params-proto
```

To reuse the S3 code mount, you can copy and pasting the `S3Mount` config from the AWS tutorial into this `.jaynes.yml` config, to replace the existing mount. Make sure that you follow the [AWS tutorial first](02_ec2_docker_guide).



### Part 2: Double-Check Your Environment Variables

you need to have these in your [~/.profile](file://~/.profile).

```bash
#~/.profile

# environment variables for Google Compute Engine
export GOOGLE_APPLICATION_CREDENTIALS=$HOME/.gce/<your-project>.json
export JYNS_GCP_PROJECT=<your-project-id-1234>
export JYNS_GCP_BUCKET=<your-bucket-name>
```

### Part 3: Docker Image

We include an example docker image in the [./docker/Dockerfile](./docker) file. You need to install `jaynes` via `RUN pip install jaynes` in the docker image, to make the jaynes entry script available.

### Part 4: Launch!

Now the launch is as simple as running

```bash
python launch_entry.py
```

Remember, turn on the  `verbose=True` flag, to see the script being generated and details of the request.

### Common Errors

- error: **name already exists**: This means that the name you are using already exists as an VM instance. You should use a different instance name.







## Config Examples and Values

Here is an example configuration for launching on GCE:

```yaml
launch: !ENV
    type: gce
    launch_dir: /home/ec2-user/jaynes-mounts
    project_id: "{env.JYNS_GCE_PROJECT}"
    zone: us-east1-b
    image_project: deeplearning-platform-release
    image_family: pytorch-latest-gpu
    instance_type: n1-standard-1
    accelerator_type: 'nvidia-tesla-k80'
    accelerator_count: 1
    preemptible: true
    terminate_after: true
```

For the `instance_type`, you can only attach GPUs to [general-purpose N1 VMs](https://cloud.google.com/compute/docs/general-purpose-machines#n1_machines) or [accelerator-optimized A2 VMs](https://cloud.google.com/compute/docs/accelerator-optimized-machines#a2_machines). GPUs are not supported by other machine families. 

### general purpose machine types 

The cpu count comes in powers of 2:

| Machine types    | vCPUs1 | Memory (GB) |
| :--------------- | :----- | :---------- |
| `n1-standard-1`  | 1      | 3.75        |
| `n1-standard-2`  | 2      | 7.50        |
| `n1-standard-4`  | 4      | 15          |
| `n1-standard-8`  | 8      | 30          |
| `n1-standard-16` | 16     | 60          |
| `n1-standard-32` | 32     | 120         |
| `n1-standard-64` | 64     | 240         |
| `n1-standard-96` | 96     | 360         |

1. A vCPU is implemented as a single hardware Hyper-thread on one of the available [CPU platforms](https://cloud.google.com/compute/docs/cpu-platforms).
2. Persistent disk usage is charged separately from [machine type pricing](https://cloud.google.com/compute/vm-instance-pricing).

For the `accelerator_type`, you can choose between the following gpus:

| value           | Details    |
| --------------- | ---------- |
| `nvidia-tesla-t4` | NVIDIA® T4 |
|`nvidia-tesla-t4-vws` | NVIDIA® T4 Virtual Workstation with NVIDIA® GRID® |
|`nvidia-tesla-p4` | NVIDIA® P4 |
|`nvidia-tesla-p4-vws` | NVIDIA® P4 Virtual Workstation with NVIDIA® GRID® |
|`nvidia-tesla-p100` | NVIDIA® P100 |
| `nvidia-tesla-p100-vws` | NVIDIA® P100 Virtual Workstation with NVIDIA® GRID® |
| `nvidia-tesla-v100` | NVIDIA® V100 |
| `nvidia-tesla-k80`| NVIDIA® K80  |

### accelerator optimized A2 types

comes in a 12:1 vCPU/A100 ratio. A2 VMs are only available on the [Cascade Lake platform](https://cloud.google.com/compute/docs/cpu-platforms).

| Machine types    | vCPUs1 | Memory (GB) |
| :--------------- | :----- | :---------- |
| `a2-highgpu-1g`  | 12     | 85          |
| `a2-highgpu-2g`  | 24     | 170         |
| `a2-highgpu-4g`  | 48     | 340         |
| `a2-highgpu-8g`  | 96     | 680         |
| `a2-megagpu-16g` | 96     | 1360        |



## Pricing

### NVIDIA GPUs

| **Model**                                                    | **GPUs** | **GPU memory** | **GPU price (USD)** | **Preemptible GPU price (USD)** | **1 year commitment price (USD)** | **3 year commitment price (USD)** |
| :----------------------------------------------------------- | :------- | :------------- | :------------------ | :------------------------------ | :-------------------------------- | :-------------------------------- |
| [NVIDIA® A100](https://www.nvidia.com/en-us/data-center/a100/) | 1 GPU    | 40 GB HBM2     | \$2.933908 per GPU   | \$0.8801724 per GPU              | \$1.84836204 per GPU               | \$1.0268678 per GPU                |
| [NVIDIA® Tesla® T4](https://www.nvidia.com/en-us/data-center/tesla-t4/) | 1 GPU    | 16 GB GDDR6    | \$0.35 per GPU       | \$0.11 per GPU                   | \$0.220 per GPU                    | \$0.160 per GPU                    |
| [NVIDIA® Tesla® P4](https://www.nvidia.com/en-us/deep-learning-ai/inference-platform/hpc/) | 1 GPU    | 8 GB GDDR5     | \$0.60 per GPU       | \$0.216 per GPU                  | \$0.378 per GPU                    | \$0.270 per GPU                    |
| [NVIDIA® Tesla® V100](https://www.nvidia.com/en-us/data-center/tesla-v100/) | 1 GPU    | 16 GB HBM2     | \$2.48 per GPU       | \$0.74 per GPU                   | \$1.562 per GPU                    | \$1.116 per GPU                    |
| [NVIDIA® Tesla® P100](https://www.nvidia.com/object/tesla-p100.html) | 1 GPU    | 16 GB HBM2     | \$1.46 per GPU       | \$0.43 per GPU                   | \$0.919 per GPU                    | \$0.657 per GPU                    |
| [NVIDIA® Tesla® K80](https://www.nvidia.com/en-gb/data-center/tesla-k80/) | 1 GPU    | 12 GB GDDR5    | \$0.45 per GPU       | \$0.135 per GPU                  | \$0.283 per GPU                    | $0.92 per GPU |

### NVIDIA® GRID® Virtual Workstation GPUs

| **Model**                                                    | **GPUs** | **GPU memory** | **GPU price (USD)** | **Preemptible GPU price (USD)** | **1 year commitment price (USD)** | **3 year commitment price (USD)** |
| :----------------------------------------------------------- | :------- | :------------- | :------------------ | :------------------------------ | :-------------------------------- | :-------------------------------- |
| [NVIDIA® Tesla® T4 Virtual Workstation](https://www.nvidia.com/en-us/design-visualization/technologies/virtual-gpu/) | 1 GPU    | 16 GB GDDR6    | \$0.55 per GPU       | \$0.31 per GPU                   | \$0.42 per GPU                     | \$0.36 per GPU                     |
| [NVIDIA® Tesla® P4 Virtual Workstation](https://www.nvidia.com/en-us/design-visualization/technologies/virtual-gpu/) | 1 GPU    | 8 GB GDDR5     | \$0.80 per GPU       | \$0.416 per GPU                  | \$0.578 per GPU                    | \$0.47 per GPU                     |
| [NVIDIA® Tesla® P100 Virtual Workstation](https://www.nvidia.com/en-us/design-visualization/technologies/virtual-gpu/) | 1 GPU    | 16 GB HBM2     | \$1.66 per GPU       | \$0.63 per GPU                   | \$1.119 per GPU                    | \$0.857 per GPU                    |





## Further Readings on GCE VM with Accelerators

To create a VM with attached GPUs, complete the following steps:

1. Create the VM. The method used to create a VM depends on the GPU model.
   - To create a VM with A100 GPUs, see [Creating VMs with attached GPUs (A100 GPUs)](https://cloud.google.com/compute/docs/gpus/create-vm-with-gpus#create-new-gpu-vm-a100).
   - To create a VM with any other available model, see [Creating VMs with attached GPUs (other GPU types)](https://cloud.google.com/compute/docs/gpus/create-vm-with-gpus#create-new-gpu-vm).
2. For the VM to use the GPU, you need to [install the GPU driver on your VM](https://cloud.google.com/compute/docs/gpus/install-drivers-gpu).
3. If you enabled NVIDIA® GRID virtual workstations, [install GRID® drivers for virtual workstations](https://cloud.google.com/compute/docs/gpus/install-grid-drivers).