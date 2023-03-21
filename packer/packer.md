# HPC AMI example with Packer
In this tutorial you'll install Packer and use it to build a custom Amazon Machine Image (AMI) containing an HPC application, [LAMMPS](https://www.lammps.org/), and en example benchmark test. You will then launch an new instance from AMI and run the benchmark test.

## Install Hashicorp Packer on your Cloud9 IDE (or on your laptop)
If you are running on a Cloud9 instance, execute the following instructions in a terminal to install Packer. Alternatively, you can install Packer on your laptop by following the instructions at [hashicorp.com](https://developer.hashicorp.com/packer/downloads).

```bash
sudo yum install -y yum-utils shadow-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
sudo yum -y install packer
```

## Create a project structure
Execute the following commands to set up a project structure.
```bash
mkdir -p lammps-ami-packer/scripts
mkdir lammps-ami-packer/resources
cd lammps-ami-packer
touch image.json
touch resources/oneAPI.repo
touch resources/run-example.sh
touch scripts/setup-oneapi-repo.sh
touch scripts/install-oneapi.sh
touch scripts/install-lammps.sh
touch scripts/install-example.sh
```
You will step through and mannually populate each of these files to control the various steps of the **provisioning** and **configurations** stages of the AMI creation process.

## Add the Intel oneAPI repository information
Copy the following contents into your **resources/oneAPI.repo** file.
```text
[oneAPI]
name=Intel(R) oneAPI repository
baseurl=https://yum.repos.intel.com/oneapi
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://yum.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB
```

This file is used to set up a yum repository in the builder instance, so that it can install the Intel oneAPI compilers and tools using the yum package manager.

## Install the oneAPI repository and do a system update
Copy the following contents to your **scripts/setup-oneapi-repo.sh** file.
```bash
#!/bin/bash
sudo yum -y install deltarpm environment-modules
# Add the Intel oneAPI repository.
sudo cp -p /tmp/oneAPI.repo /etc/yum.repos.d/
sudo chmod 744 /etc/yum.repos.d/oneAPI.repo
sudo yum -y update 
sudo reboot
```

This script installs the repository definition above into the correct location and preforms an update of all packages on the system.


## Install the Intel oneAPI compilers and tools
Copy the following contents to your **scripts/install-oneapi.sh** file
```bash
#!/bin/bash

### Install some of the Intel oneAPI compilers, libraries and tools, including VTune.
sudo yum -y install \
gcc \
"kernel-devel-uname-r == $(uname -r)" \
kernel-headers

sudo yum -y group install "Development Tools"
sudo yum -y install \
intel-oneapi-compiler-dpcpp-cpp-and-cpp-classic \
intel-oneapi-mkl \
intel-oneapi-mkl-devel \
intel-oneapi-mpi \
intel-oneapi-mpi-devel \
#intel-oneapi-compiler-dpcpp-cpp \
#intel-oneapi-compiler-fortran \
#intel-oneapi-vtune 


# Install Modules
sleep 10
echo "source /usr/share/Modules/init/bash" >> ${HOME}/.bashrc
source ${HOME}/.bashrc

# Set up Modules to find the oneAPI tools we just installed.
sudo /opt/intel/oneapi/modulefiles-setup.sh
echo "module use /opt/intel/oneapi/modulefiles" >> ${HOME}/.bashrc
source ${HOME}/.bashrc
```

This script installs some of the Intel oneAPI compilers, libraries and tools, and sets up the modules environment.


## Install LAMMPS
Copy the following contents to your **scripts/install-lammps.sh** script.
```bash
#!/bin/bash
sudo yum install -y cmake3 zlib libjpeg libjpeg-devel nasm yasm
module avail
module load icc mpi mkl

sudo mkdir /opt/lammps
sudo chown -R ec2-user:ec2-user /opt/lammps
cd /opt/lammps
git clone https://github.com/lammps/lammps.git
cd lammps
mkdir -p build
cd build
cmake3 ../cmake \
  -DPKG_MANYBODY=yes \
  -DPKG_REPLICA=yes \
  -DPKG_MOLECULE=yes \
  -DPKG_KSPACE=yes \
  -DBUILD_MPI=yes \
  -DBUILD_OMP=yes \
  -DLAMMPS_MACHINE=AWS \
  -DCMAKE_C_COMPILER=`which mpicc` \
  -DCMAKE_CXX_COMPILER=`which mpicxx` \
  -DWITH_JPEG=yes \
  -DWITH_GZIP=yes
make -j 128

```

These commands download the LAMMPS source code and compile it using the Intel oneAPI C++ compiler.

## Add the example runscript
Copy the following contents to your **resources/run-example.sh** script.
```bash
#!/bin/bash
module load icc mpi mkl
cd /opt/lammps/bm
which mpirun
ldd /opt/lammps/build/lmp_AWS
export OMP_NUM_THREADS=1
export NP=$(grep processor /proc/cpuinfo | wc -l)
mpirun -n $NP /shared/lammps/build/lmp_AWS -in cu.in 
```

This script can be executed on an instance running the resultant AMI to run a small example LAMMPS text simulation.


## Install the example
Copy the following contents to your **scripts/install-example.sh** script.
```bash
#!/bin/bash
mkdir /opt/lammps/bm
cd /opt/lammps/bm
curl -O https://www.lammps.org/inputs/in.eam.txt

## Remove the last line 
sed '$d' in.eam.txt > cu.in
## Add new parameters
echo 'dump            1 all atom 5000 cu_dump.lammpstrj' >> cu.in
echo 'run             40000' >> cu.in
cp /opt/lammps/bm/Cu_u3.eam .

cp -p /tmp/run-example.sh /opt/lammps/bm/
chmod 744 /opt/lammps/bm/run-example.sh
```

These commands download and configure the example problem and install the example run script.

## Set up the Packer template
Copy the following contents to your **image.json** file.
```json
{
    "variables": {
        "iam_instance_role": "{{env `INSTANCE_ROLE`}}",
        "region": "{{env `DEFAULT_AWS_REGION`}}",
        "ami_name": "LAMMPS oneAPI",
        "ami_description": "LAMMPS built with Intel oneAPI compilers on Amazon Linux 2."
    },
    "builders": [{
        "type": "amazon-ebs",
        "region": "{{user `region`}}",
        "source_ami_filter": {
            "filters": {
                "virtualization-type": "hvm",
                "name": "amzn2-ami-*",
                "root-device-type": "ebs"
            },
            "owners": ["137112412989"],
            "most_recent": true
        },
        "instance_type": "c6i.32xlarge",
        "ssh_username": "ec2-user",
        "iam_instance_profile": "{{user `iam_instance_role`}}",
        "ami_virtualization_type": "hvm",
        "ebs_optimized": true,
        "ami_name": "{{user `ami_name`}} {{timestamp}}",
        "ami_description": "{{user `ami_description`}}",
        "launch_block_device_mappings": [{
            "device_name": "/dev/xvda",
            "volume_size": 20,
            "volume_type": "gp3",
            "delete_on_termination": true
        }],
        "tags": {
            "OS_Version": "Amazon Linux 2",
            "Release": "{{timestamp}}",
            "Name": "LAMMPS oneAPI",
            "Base_AMI_Name": "{{ .SourceAMIName }}"
        }
    }],
    "provisioners": [{
        "type": "file",
        "source": "./resources/oneAPI.repo",
        "destination": "/tmp/oneAPI.repo"
    }, {
        "type": "file",
        "source": "./resources/run-example.sh",
        "destination": "/tmp/run-example.sh"
    }, {
        "type": "shell",
        "script": "./scripts/setup-oneapi-repo.sh",
        "expect_disconnect": true,
        "valid_exit_codes": [0, 1, -1],
        "pause_before": "10s"
    }, {
        "type": "shell",
        "script": "./scripts/install-oneapi.sh",
        "valid_exit_codes": [0, 1, -1],
        "pause_before": "10s"
    }, {
        "type": "shell",
        "script": "./scripts/install-lammps.sh"
    }, {
        "type": "shell",
        "script": "./scripts/install-example.sh"
    }],
    "post-processors": [
        [{
            "output": "manifest.json",
            "strip_path": true,
            "type": "manifest"
        }]
    ]
}
```

This file controls the image building process and copies the **resources** files and runs the **scripts** you defined above.

This template has a **provisoning stage**, which defines the characteristice of a temporary builder instance, and a set of steps in the **configuration stage**, that copies the files in the resources directory, and executes a set of scripts.


## Build an AMI using Packer

Execute the following commands to initialise the Packer project, valudate the template, and build the AMI.

```bash
export DEFAULT_AWS_REGION=$(aws configure get region)
packer validate image.json
packer build image.json
```


## Launch an instance from the AMI

Once you've successfully built the image, you can navigate to the [AMI section of the AWS console](https://console.aws.amazon.com/ec2/home#Images) to see your newly built AMI, named "LAMMPS oneAPI".

Launch a new instance from this AMI: 
1. Name it "LAMMPS-oneAPI-test".
2. Choose **c6i.4xlarge**.
3. Choose your preferred key pair.

## Connect to the instance and run the test LAMMPS job

1. Locate your newly running instance in the [instances section of the AWS Console](https://console.aws.amazon.com/ec2/home#Instances:instanceState=running), select it and click on "Connect" to ssh to your new instance.
2. Connect with ssh as ec2-user, specifying the same key that you used when you launched the instance.
3. Now run the following commands to run the LAMMPS test job.
```bash
cd /opt/lammps/bm
./run-example.sh
```

You should see your job running and upon sucessful completion you should see a line resembling the following at the end of the output:

```text
Total wall time: 0:02:33
```

You can use the following command to report the simulation performance from the run:

```bash
grep Performance log.lammps
```

You should see output that resembles the following:

```text
Performance: 112.446 ns/day, 0.213 hours/ns, 260.292 timesteps/s, 8.329 Matom-step/s
```
