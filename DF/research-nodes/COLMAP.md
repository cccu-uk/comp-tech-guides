# COLMAP guide for Computing research on the forensic cluster

## Start
- SSH into the server
- Open another terminal tab and use `cd` to navigate to your local machine directory containing your project files
- Copy your project files to your home directory on the server using `rsync`. Use the `-P` flag for progress/partial transfers.
- It may be a good idea to manage your project using Git where possible. 

Consider using bash scripts to help automate/repeat your jobs.

## Example COLMAP setup:

- Download and unzip the example dataset https://colmap.github.io/datasets.html using `curl -L -O https://github.com/colmap/colmap/releases/download/3.11.1/south-building.zip` ... `unzip south-building.zip`

- Pull the COLMAP image from dockerhub using aptainer `apptainer build colmap.sif docker://colmap/colmap:20260929.8466` (you then get a .SIF file)

## Running COLMAP on the example dataset

> **Use tmux for this**

### feature extraction:
`mkdir -p outputs && apptainer exec --userns --nv --bind $(pwd):/workspace colmap.sif colmap feature_extractor --database_path /workspace/outputs/database.db --image_path /workspace/south-building`

### feature matching:
`apptainer exec --userns --nv --bind $(pwd):/workspace colmap.sif colmap exhaustive_matcher --database_path /workspace/outputs/database.db`

### sparse reconstruction/mapper:
`mkdir -p outputs/sparse && apptainer exec --userns --nv --bind $(pwd):/workspace colmap.sif colmap mapper --database_path /workspace/outputs/database.db --image_path /workspace/south-building --output_path /workspace/outputs/sparse`

