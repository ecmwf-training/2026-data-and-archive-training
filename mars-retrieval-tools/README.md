# Follow the steps below to prepare your environment before running the examples.

---
## Authentication
If you are running this outside the HPC/ECS environment, you need to create an .ecmwfapirc file.
Place this file in your $HOME directory and populate it with the content from your [ecmwf api key](https://api.ecmwf.int/v1/key/)


## 1. If Running on the HPC/ECS

If you're working on the ECMWF HPC system, load the appropriate Python module:

```bash
module load python3
```
*If you're not using the HPC, you can skip this step.*



## 2. Creating a Python Virtual Environment with `venv`

```bash
envname=.venv

# Create a virtual environment
python3 -m venv $envname

# Activate it
source $envname/bin/activate      # macOS/Linux

# Install dependencies
pip install -r requirements.txt

# If using Jupyter notebooks, register your environment as a kernel for ipykernel
python3 -m ipykernel install --user --name=$envname

# Optional: Deactive the environment after you are done using it
deactivate

```

## 3. Select the Kernel in Your Notebook
Select the kernel with the envname that you just created to run your notebook.
It may take some time to load.

---

## Extras

### Check Installed Kernels
```bash
jupyter kernelspec list
```

### Uninstall a Kernel
```bash
jupyter kernelspec uninstall <envname>
```