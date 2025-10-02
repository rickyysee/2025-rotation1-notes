# Rotation with Dr. Titus Brown using FARM Cluster
### Getting an Account on FARM
* `cd` into the `.ssh` folder or `mkdir -p ~/.ssh` to make one
* `ssh-keygen` creates a keyfile pair, a public and private
* Apply for an account on [this page](https://hippo.ucdavis.edu/Farm/myaccount) and upload public ssh key

### Connecting to a remote computer
```
ssh -i ~/.ssh/farm rcantua@farm.cse.ucdavis.edu
```
> `exit` will terminate the session

### Starting a `tmux` Session
**When you log in to FARM** (i.e. from the log in node), you can use `tmux` to allow commands to continue to run even if you close your connection.
```
tmux new -s dib
```
> This creates a new session with the name `dib`.

You can detach a session with `Ctrl+b`,`d`.

```
tmux ls
```
> This will list all tmux sessions.

```
tmux a -t dib
```
> This will allow you to *target* session dib.

### ~~Installing Conda~~
We will install Miniconda, which includes Conda, python, and some other dependencies.
```
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh
```

### ~~Activating Miniconda~~
```
source ~/.bashrc
```

### ~~Configuring Channels~~
Channels dictate where packages are installed from and are checked in terms of priority. Configure channels so that `conda-forge` is the highest priority as so:
```
conda config --add channels defaults
conda config --add channels bioconda
conda config --add channels conda-forge
```
Also install the mamba alternative frontend to conda.
```
conda install -y "mamba>=0.22.1"
```

### Loading Conda Module
```
module load conda
```
> This will load HPC@UCD's central conda installation.

### Creating an Environment
An environment is a self-contained sector of a computer with packages and dependencies that do not communicate with other environments.
```
mamba create -y -n dib_rotation "python=3.8"
```
> This creates the `dib_rotation` environment with Python 3.8, which will be used for some software in this rotation.

To activate this environment:
```
conda activate dib_rotation
```
> The environment will be displayed by the prompt, `(base)` or `(dib_rotation)`

Install sourmash, a program created by the DIB lab that will be used later.
```
mamba install -y sourmash
```

### Deactivating and Exiting
You can use `conda deactivate` to return to the base environment.

When you log out of FARM with `exit`, end a `tmux` or `screen` session, or an `srun` job ends, your environment will automatically be exited.

### Accessing a Compute Node
When you login to FARM, you'll be on a `login` node, which is shared by everyone and can get backed up if used to run big processes.

We can use the job scheduler to get compute resources for our session.
```
srun -p bmm -J rotation -t 5:00:00 --mem=10G --pty bash
```
> `srun` uses the job scheduler `SLURM` 
> `-p` specifies the job queue we want to use, specific to FARM accounts
> `-J rotation` is the job name
> `-t` denotes how long we want the computer for
> `--mem` specifies how much memory we want the computer to have (10G = 10 Gigabytes)
> `--pty bash` specifies that we want the linux shell to be bash

Your prompt should change from `username@farm` to `username@bm1` or some other number.

The files you see in the login node and job computers are the same since they write to the same hard drives.

Activate your conda environment once you're in the `srun` session.

### Leaving and Reattaching `tmux` Session
Use the keys `Ctrl-b`,`d` to exit
```
tmux attach
```
