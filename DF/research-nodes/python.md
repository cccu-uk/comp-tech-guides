# Python guide for Computing research on the forensic cluster

## Start

- SSH into the server
- Open another terminal tab and use `cd` to navigate to your local machine directory containing your project files
- Copy your project files to your home directory on the server using `rsync`. Use the `-P` flag for progress/partial transfers.

## Pulling python image

- Pull the python image from dockerhub using aptainer `apptainer build python-3.15.1.sif docker://python:3.15.1` (you then get a .SIF file)

## Using the python image

> **Use tmux for running jobs**

- CD into your working directory (this will be mounted to the container as `/workspace`)
- open a shell `apptainer shell --userns --nv --bind $(pwd):/workspace python-3.15.1.sif`
- while inside the container, to install packages use the --user flag so that they persist when you exit/re-enter the container. This stores the packages in your home directory on the server: `pip install --user scikit-learn`
- Execute your python jobs from inside the container. Relative paths will work but the container absolute path to your project folder will begin with `/workspace`. Example command to execute a script `python3 /workspace/example.py`
