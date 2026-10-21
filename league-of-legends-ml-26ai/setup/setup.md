# Prepare Your Workspace

## Introduction

Every model in this workshop runs in a Python 3.12 environment with pinned library versions. You can build that environment on your own computer (Option A, recommended) or inside an OCI Data Science notebook session (Option B). Both options end the same way: a terminal where `python` is Python 3.12, a clone of the workshop repository, and a passing import check.

The earlier version of this workshop provisioned a virtual machine, a database and a notebook session with Terraform and Ansible before you could run a single cell. None of that is needed for the core path. The database arrives in Lab 9, when you use it.

Estimated Time: 20 minutes (Option A), 35 minutes (Option B)

### Objectives

In this lab, you will:

* Install `uv`, a fast Python installer that also manages Python versions
* Clone the workshop repository and create a Python 3.12 environment
* Install the pinned requirements and check them
* Optionally run the workshop smoke test
* Optionally prepare an OCI Data Science notebook session instead of your computer

### Prerequisites

This lab assumes you have:

* Git installed
* 10 GB of free disk space and an internet connection
* For Windows: PowerShell 5.1 or later, or WSL2 (the commands below work in both; WSL2 behaves like Linux)

## Task 1: Install uv

[uv](https://docs.astral.sh/uv/) downloads the right Python version for you, so you do not have to touch the Python that came with your operating system.

<if type="linux">

1. Install `uv` (this also works on macOS).

    ```bash
    <copy>
    curl -LsSf https://astral.sh/uv/install.sh | sh
    </copy>
    ```

2. Open a new terminal and check the installation.

    ```bash
    <copy>
    uv --version
    </copy>
    ```

</if>

<if type="windows">

1. In PowerShell, install `uv`.

    ```powershell
    <copy>
    powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
    </copy>
    ```

2. Open a new PowerShell window and check the installation.

    ```powershell
    <copy>
    uv --version
    </copy>
    ```

</if>

## Task 2: Clone the repository and create the environment

1. Clone the repository and move into the workshop folder. Every command in the labs runs from this folder unless a lab says otherwise.

    ```bash
    <copy>
    git clone -b livelabs-revamp-2026 https://github.com/jasperan/leagueoflegends-optimizer.git
    cd leagueoflegends-optimizer/lol-ml-26ai
    </copy>
    ```

2. Create a Python 3.12 environment in `.venv` and activate it.

    <if type="linux">

    ```bash
    <copy>
    uv venv --python 3.12 .venv
    source .venv/bin/activate
    </copy>
    ```

    </if>

    <if type="windows">

    ```powershell
    <copy>
    uv venv --python 3.12 .venv
    .venv\Scripts\Activate.ps1
    </copy>
    ```

    </if>

3. Install the pinned requirements. This downloads PyTorch, AutoGluon and the other libraries, which are several gigabytes together. It takes a few minutes.

    ```bash
    <copy>
    uv pip install -r requirements.txt
    </copy>
    ```

    The file pins every direct dependency to the version the workshop was tested with: AutoGluon 1.6.3, scikit-learn 1.9.1, pandas 2.3.3, PyTorch 2.10, python-oracledb 26.0.1 and others. The pins matter. Unpinned installs were the reason the earlier labs stopped working: a new NumPy broke the old pandas, and an AutoGluon model trained with one version cannot be loaded by another.

## Task 3: Check the installation

1. Print the versions of the main libraries.

    ```bash
    <copy>
    python -c "import sys, autogluon.tabular as ag, sklearn, torch, fastai, pandas, oracledb; print('python', sys.version.split()[0]); print('autogluon', ag.__version__, '| scikit-learn', sklearn.__version__, '| torch', torch.__version__, '| fastai', fastai.__version__, '| pandas', pandas.__version__, '| oracledb', oracledb.__version__)"
    </copy>
    ```

2. The output should look like this. A patch version of Python (`3.12.x`) may differ.

    ```text
    python 3.12.2
    autogluon 1.6.3 | scikit-learn 1.9.1 | torch 2.10.0 | fastai 2.8.7 | pandas 2.3.3 | oracledb 26.0.1
    ```

## Task 4: Run the smoke test (optional)

The repository includes a smoke test that runs the whole core path on a small sample that ships with the repository: unit tests, notebooks 01 to 04, training and exporting the serving model, and live scoring against a mock Live Client. It needs no download, no API key and no cloud account.

1. Run it. It takes three to five minutes.

    ```bash
    <copy>
    bash scripts/smoke.sh
    </copy>
    ```

    On Windows, run it in WSL2 or Git Bash. In plain PowerShell, skip this task: you run the same steps one by one in the next labs.

2. The last line must read `SMOKE TEST PASSED`.

## Task 5: Use an OCI Data Science notebook session instead (Option B)

Choose this option if you cannot install software on your computer, or if you want to train on more CPU cores. A notebook session is a JupyterLab workspace on a compute instance that Oracle manages. It bills while it is active, so you delete it in the Clean Up lab.

> **Note:** The OCI console steps below follow the current OCI documentation. They were not run end to end while this workshop was written, because that requires creating cloud resources. If a label differs in your console, use the linked documentation pages. The Python part (the pinned install and the checks) was tested on a laptop with the same Python version.

1. Make sure your user can create a Data Science project and notebook session in a compartment, and that the compartment's network allows the notebook to reach the internet (the default networking option does, and it also gives access to Object Storage). See [Creating a Notebook Session](https://docs.oracle.com/en-us/iaas/Content/data-science/using/create-notebook-sessions.htm) for the required policies and networking.

2. In the OCI console, open the navigation menu, click **Analytics & AI**, and then click **Data Science** under **Machine Learning**.

3. Create a project: click **Create project**, name it `lol-ml`, and click **Create**.

4. In the project, click **Create notebook session**. Choose a compartment, a name such as `lol-ml-session`, a **Flex** shape (for example **VM.Standard.E5.Flex** with 4 OCPUs and 32 GB of memory) and a block storage size of at least 100 GB. Keep **Default networking**. Click **Create** and wait until the state is **Active**.

5. Click **Open** to start JupyterLab. In the **Launcher** tab, click **Terminal**.

6. Install a current service conda environment for Python 3.12. The workshop was tested on Python 3.12, and the OCI list of active environments contains `generalml_p312_cpu_x86_64_v4` (General Machine Learning for CPUs on Python 3.12). Environment Explorer in the **Launcher** shows which versions exist in your region; use the slug it shows if it differs.

    ```bash
    <copy>
    odsc conda install --slug generalml_p312_cpu_x86_64_v4
    </copy>
    ```

    Answer the prompt that asks about changing the version of the environment by pressing **Enter**. Do not install into the default base environment: it is not kept when the session is reactivated.

7. Activate the environment, clone the repository and install the pinned requirements on top.

    ```bash
    <copy>
    conda activate /home/datascience/conda/generalml_p312_cpu_x86_64_v4
    git clone -b livelabs-revamp-2026 https://github.com/jasperan/leagueoflegends-optimizer.git
    cd leagueoflegends-optimizer/lol-ml-26ai
    pip install -r requirements.txt
    python -m ipykernel install --user --name lolml --display-name "lolml (Python 3.12)"
    </copy>
    ```

8. Run the check from Task 3 in this terminal. If `pip` reports a conflict with a package that the conda environment brought along, create a clean environment instead: `python -m venv ~/lolml-venv`, activate it, and run `pip install -r requirements.txt` again.

9. When you open notebooks from the `notebooks` folder, select the kernel **lolml (Python 3.12)** (**Kernel**, **Change Kernel**).

## Learn More

* [uv documentation](https://docs.astral.sh/uv/)
* [OCI Data Science: Creating a Notebook Session](https://docs.oracle.com/en-us/iaas/Content/data-science/using/create-notebook-sessions.htm)
* [OCI Data Science: Installing Conda Environments in a Notebook Session](https://docs.oracle.com/en-us/iaas/Content/data-science/using/conda_installing.htm)
* [Active service conda environments](https://github.com/oracle-samples/oci-data-science-ai-samples/blob/main/conda-environments/service-conda-environments.md)

## Acknowledgements

* **Author** - Ignacio Martinez, Oracle
* **Last Updated By/Date** - Ignacio Martinez, October 2026
