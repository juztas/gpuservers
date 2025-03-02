# iBanks NVIDIA powered GPU cluster

## Nodes and Access

The head node `ibanks.hep.caltech.edu` is running the nfs home and data server.
It has a public (regular network) and a private (10G) IP.

Worker nodes are a follows, in chronological order of creation
* ~~`passed-pawn-klmx.hep.caltech.edu` is a cocolink server with 200G local disk, and runs 8  NVidia Titan X (Pascal)~~
* `culture-plate-sm.hep.caltech.edu` is a Supermicro server with 2T of local SSD, and runs 8 NVidia GeForce GTX 1080 (old setup, will not work a priori with new accounts)
* `imperium-sm.hep.caltech.edu` is a Supermicro server with 2T of local SSD, and runs 8 NVidia GeForce GTX 1080
* `flere-imsaho-sm.hep.caltech.edu` is a Supermicro server with 2T of local SSD, and runs 6 NVidia Titan Xp (Pascal)
* `mawhrin-skel-sm.hep.caltech.edu` is a Supermicro server with 2T of local NVME running 2 NVidia GeForce GTX Titan X

All server have a public (regular network) and a private (10G) IP.
SSH key is the only authentication. Please let the admins know if you need help setting this up.
 
## Credits

If you are a user of the cluster, we are happy to help make progress.
When producing publication or public presentation, please be so kind as to warn Prof. Spiropulu and Dr. Vlimant, just for accounting purpose.
Please include the following latex acknowledgement for support
```latex
Part of this work was conducted at  "\textit{iBanks}", the AI GPU cluster at Caltech. We acknowledge NVIDIA, SuperMicro  and the Kavli Foundation for their support of "\textit{iBanks}".
```

## Credentials

If you are not familiar on how to create an ssh key, from a remote client (your laptop) run the following command
<pre>
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
</pre>
and place the content of the public key in the .ssh/authorize_keys on the ibanks nfs home directory.

### Data Storage

The home directory should be used for software and although there is room, please prevent from putting too much data within your home directory.

The `/bigdata/` volume is mounted on all nodes. It is a 20TB raid array mounted over nfs. Please use the `/bigdata/shared/` directory and contact the admins if in the need for private directory.

The `/data/` volume is mounted on some nodes, not all on SSD. This is the prefered temporary location for data needed for intensive I/O.

The `/imdata/` volume is a ramdisk of 40G with very high throughput, but utilizing the RAM of the machine. Please use this in case of need of very high i/o, but clean the space tightly, as this will use the node memory. There is a 2-day-last-access retention policy on it.

The `/t2data/` path is the home directory on the caltech Tier2.

The `/mnt/hadoop/` path is the readonly access to the full caltech Tier2 storage.

The `/storage/group/gpu` path is a 120TB CEPH volume that can be used similarly to bigdata.

### Setup

It is important to note that I/O on the nfs mounted volume is not as efficient as with local disk, so please use care and monitor performance of your applications.

For ipython, the following directory better be local
<pre>
mkdir /tmp/$USER/ipython -p
ln -s /tmp/$USER/ipython .ipython
</pre>

For cuda, the same applies to
<pre>
mkdir -p /tmp/$USER/cuda/
export CUDA_CACHE_PATH=/tmp/$USER/cuda/      
</pre>
It is recommended to have `export CUDA_CACHE_PATH` in your login file.

To use only a selected GPU, run `nvidia-smi` or `gpustat` to see GPU utilization, then set `export CUDA_VISIBLE_DEVICES=n` to a the index of the GPU you want to use.
In python one can either set the environment variable or use `import setGPU` (gets one device automatically).

## Software

### Singularity

All the software is provided with singularity images located in `/bigdata/shared/Software/singularity/ibanks/`
Configuration of the images is located at https://github.com/cmscaltech/gpuservers/tree/master/singularity and in `/bigdata/shared/Software/gpuservers/singularity/`

| image | description |
|-------|-------------|
| legacy.simg | A fixed image with the software already installed on ibanks nodes |
| edge.simg | An image with many of the useful libraries, with the latest versions | 

Let admins know of any missing library that can be put in the image. A build service will be setup later.


To start a shell in an cutting edge image
<pre>
/bigdata/shared/Software/gpuservers/singularity/run.sh
</pre>
or to start with a given image
<pre>
/bigdata/shared/Software/gpuservers/singularity/run.sh /bigdata/shared/Software/singularity/ibanks/legacy.simg 
</pre>

To build an image, first make sure that there are not an existing image that is usable, or extendable for your purpose. There are example of image specifications under the [https://github.com/cmscaltech/gpuservers/tree/master/singularity][singularity directory], to create your `specification.singularity` file
To build the image `myimage.simg` from the spec
<pre>
/bigdata/shared/Software/gpuservers/singularity/build myimage.simg specification.singularity
</pre>
if you make changes to existing image, please provide suggestion via a pull request modifying the specification file.

### Tensorflow

Tensorflow is greedy in using GPUs and it is mandatory to use `export CUDA_VISIBLE_DEVICES=n` (where n is the index of a device, or coma separated index) to use only a selected device, if not explicitly controlled within the application.
In python, please use `import setGPU` that selects automatically the next available GPU.

### Jupyter Hub

Work in progress to set this up properly on the cluster.

### Jupyter Notebook

The users can start a jupyter notebook server on each machine using either

<pre>
/bigdata/shared/Software/gpuservers/jupyter/start_S.sh
</pre>
 
 to start a notebook with the latest singularity image. Or 

<pre>
/bigdata/shared/Software/gpuservers/jupyter/start_S.sh /bigdata/shared/Software/singularity/ibanks/legacy.simg
</pre>
if a given image.

This will provide back a url to which you can connect, including an authentication token, that changes each time you restart the jupyter server. You should keep this token private, but can also share momentarily to let other people edit your notebooks ; beware anyone with the token is "you".

The port that is assigned to you is defined in `/bigdata/shared/Software/jupyter/ports` if you are not in there, please contact an admin.

### MPI

mpi is available within nodes (but not yet accross nodes). One needs to first some env variables
<pre>
MPI_PATH=/opt/openmpi-3.1.0
PATH=$MPI_PATH/bin/:$PATH
LD_LIBRARY_PATH=$MPI_PATH/lib/:$LD_LIBRARY_PATH
OPAL_PREFIX=$MPI_PATH
</pre>

To run a program with mpi
<pre>
mpirun -np 3 nvidia-smi
</pre>

To run a program using singularity with mpi
<pre>
mpirun -np 3 singularity exec -B /bigdata --nv /bigdata/shared/Software/singularity/ibanks/edge-n.simg python3 /bigdata/shared/Software/mpi/mpi4py-examples/03-scatter-gather
</pre>
