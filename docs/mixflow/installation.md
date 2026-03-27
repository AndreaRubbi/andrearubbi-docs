# Installation

[Home](index.md) | [Installation](installation.md) | [Running SP-FM](running.md)

---

SP-FM is currently in active development and not available as a python library. It can, however, be used through the scripts available in the github repository.

## 1. Clone the repository

```bash
git clone https://github.com/Lotfollahi-lab/SP-FM.git
cd SP-FM
```

## 2. Create and activate an environment (optional but recommended)

For example, using `conda`:

```bash
conda create -n SP-FM python=3.10 -y
conda activate SP-FM
```

You can also use `venv` or another environment manager if you prefer.

## 3. Install SP-FM in editable mode

From the repository root:

```bash
pip install -e .
```

This installs SP-FM and its Python dependencies while keeping the source editable for development.

## 4. Install Exact Environment

We provide the `requirements.txt` file that can be used to recreate exactly the environment used for our benchmarks.

```bash
pip install -r requirements.txt
```
---

Once installed, you can follow the instructions in the [Running SP-FM](running.md) page for an example training run on the Norman dataset.