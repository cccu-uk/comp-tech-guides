# AI node access and usage

## Setup summary

- The AI node itself lives in the server rack in VH1.30.
- The nodes run the NixOS flavour of Linux.
- You are responsible for your data and should back it up regularly in multiple places (Google the 3-2-1 backup strategy). **We do not back up your data.**
- The host system is a launch pad, not a place to install ML software.
- **GPU and ML work (PyTorch, COLMAP, OpenCV, CUDA code) runs inside Apptainer containers.**
- **Host Python is for light scripting and orchestration only:** moving files, looping over scenes, launching container jobs.
- Keep all your data in your home directory.
- Run long jobs inside `tmux` so they survive a dropped connection.


## AI nodes basic specification

* Tesla V100 16GB GPU
* AMD Epyc 7452 x2 (total 64 cores)
* 512GB RAM


## Generating your SSH keys so your device can connect to the AI node

1. Open a terminal on the client machine with SSH installed (installed by default on Windows)
2. If you don't already have an SSH key, use `ssh-keygen` to create one. Press enter to accept default file path/name and set an optional passphrase.
3. From your home directory (`cd ~`) enter `cat .ssh/id_XXXXX.pub` copy the full output and send this to a technician so your device can access the nodes.

*Never share your private key!!*


## Connecting with SSH

Contact technicians to get configured for accessing the servers. Once you are configured, you can access via SSH:
- Open a terminal on a system with SSH installed.
-  Use command `ssh cccu-nodeX` where X is 1, 2, or 3.


## Python on the nodes

### What host Python is for

- Small scripts that move, rename, sort or list files
- Orchestration: looping over scenes or parameter settings and launching container jobs
- Reading and writing config files (YAML, JSON), light data wrangling, and plotting small results if you've been given the packages for it
- Anything that doesn't need a GPU or compiled ML libraries

### What host Python is NOT for

- Training or running models (`torch`, `tensorflow`, `jax`)
- Anything that imports CUDA-dependent libraries
- Installing packages with compiled code (OpenCV, SciPy builds, most ML packages) via `pip`
- Installing a second Python by other means (Miniconda or Anaconda installers, `uv`-downloaded interpreters)

These generally fail on NixOS with confusing errors such as "file not found" on a binary that clearly exists, because NixOS does not have a standard system linker. Use a container instead.

### What's installed on the host

The system `python3` includes a fixed set of packages that were built for NixOS and work natively:

- numpy, pandas, pyyaml, requests, tqdm, click

If you need another package on the host, you can send a request to technicians to add it.

### Adding your own pure-Python packages

Create a virtual environment that can see the system packages:

```bash
python3 -m venv --system-site-packages /home/<username>/venvs/tools
source /home/<username>/venvs/tools/bin/activate
pip install <some-pure-python-package>
```

This works for packages that contain only Python code. If the install builds something or the import fails with a missing library error, that package belongs in a container.

Things that will not work, by design:

- `pip install <package>` outside a venv (the system Python is read-only and protected)
- `pip install --user`


## Running ML tools with Apptainer

Apptainer runs software in a self-contained image, so PyTorch, COLMAP and their CUDA libraries come with the image and don't depend on the host.

### The basic command

```bash
apptainer exec --nv --cleanenv \
    /home/<username>/pytorch.sif \
    python my_script.py --input /home/<username>/scene01
```

| Flag | What it does |
|---|---|
| `--nv` | Makes the NVIDIA GPUs available inside the container. Needed for any GPU work. |
| `--cleanenv` | Stops your host environment variables from leaking into the container. |

Notes:

- Your home directory and the current directory are mounted automatically.
- Anything the container writes **outside** a bound or home path is lost when it exits. Always write outputs to your home directory.
- If packages in `~/.local/lib/python3.*` cause conflicts inside a container, add `--env PYTHONNOUSERSITE=1`, or run with `--no-home` and bind what you need explicitly.


### Pulling an image

`apptainer pull my-pytorch.sif docker://pytorch/pytorch:<tag>`

Guidelines:

- **The GPUs are NVIDIA V100s.** Choose images built on **CUDA 12.x**. CUDA 13 images do not support this GPU generation.
- **Pin a specific tag**, not `latest`, so your results stay reproducible.
- Use official or well-known publishers only. An image can run any code as you.

### Installing extra Python packages inside a container

Create the environment **inside the container**, stored in your data directory so it persists:

```bash
apptainer exec --nv --cleanenv pytorch.sif \
    python -m venv --system-site-packages /home/<username>/venvs/my-env

apptainer exec --nv --cleanenv pytorch.sif \
    /home/<username>/venvs/my-env/bin/pip install <package>
```

Rules for container venvs:

- `--system-site-packages` lets the venv reuse the image's PyTorch and CUDA.
- A venv is tied to the image that created it. Use a **new venv per image**, and don't switch images under an existing one.


### Clear apptainer cache

Use `apptainer cache clean` every now and then to clear redundant data.


## Resource sharing and tmux

While you are the only user using a node, you do not need to consider resource sharing too much. This may change in the future.

Ensure you check the GPU is free before starting work using the `nvidia-smi` command to see active processes.

If there is more demand for server use, we will coordinate usage in a Teams chat.

- Use `tmux` for long jobs:

```bash
tmux new -s myjob      # start a session
# run your job, then detach with Ctrl-b then d
tmux attach -t myjob   # come back later
```

[**Tmux cheat sheet**](https://tmuxcheatsheet.com/)

- When a job finishes, release the GPU. Don't leave processes holding it.


## Copying files to/from the server

* [File management in the terminal](https://earthdatascience.org/courses/intro-to-earth-data-science/open-reproducible-science/bash/bash-commands-to-manage-directories-files/)
* [Rsync](https://linuxize.com/cheatsheet/rsync/)


## Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| "No such file or directory" when running a binary that exists | A native binary or wheel is running on NixOS. Run it in a container. |
| `pip install` says the environment is externally managed | Use a venv or a container. |
| `torch.cuda.is_available()` is `False` in a container | Missing `--nv`, or the image uses a CUDA version the driver doesn't support. Try a CUDA 12.x image. |
| Output files vanished after a container job | They were written outside a bound path. Write to home directory. |
| Wrong or odd package versions inside a container | Host packages leaking in. Use `--cleanenv` and `PYTHONNOUSERSITE=1`. |
| COLMAP says no CUDA / GPU unavailable | Missing `--nv`, or the image wasn't built for this GPU generation (V100 is compute capability 7.0). Ask the admin. |
| Venv stopped working after switching images | Venvs are tied to one image. Create a new one. |


## Getting help

Contact technicians. When asking for help, include the exact command you ran and the full error message.
