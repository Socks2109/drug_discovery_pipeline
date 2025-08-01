# **Targeted Molecular Docking Workflow for HIF-2α**

This document outlines the step-by-step procedure followed to perform targeted molecular docking of ligands against specific residues of the Hypoxia-Inducible Factor 2-alpha (HIF-2α) PAS-B domain. The workflow utilizes UCSF Chimera/ChimeraX for receptor preparation and residue selection, Meeko for preparing receptor PDBQT files and targeted Grid Parameter Files (GPFs), AutoGrid4 for grid map generation, and AutoDock-GPU for the docking calculations.

---

![image](https://github.com/user-attachments/assets/483554bd-d2ad-4e6b-b08f-1862796b9758)


## **1. Software and Environment Setup**

The following software and environment configuration was used:

* **Conda Environment:** A dedicated Conda environment was created to manage dependencies, specifically to resolve version compatibility issues.
    * Python version: 3.10
    * Environment name (example): `meeko_py310`
        ```bash
        # Command to create the environment (example)
        conda create -n meeko_py310 python=3.10
        conda activate meeko_py310
        ```
* **Meeko & RDKit:** Installed into the Conda environment from the `conda-forge` channel. 
    ```bash
    conda install -c conda-forge meeko rdkit
    sudo apt install -U meeko 
    ```
* **AutoDock-GPU:** The GPU-accelerated docking engine. (Assumed to be pre-installed and accessible).
* **AutoGrid4:** Part of the AutoDock4 suite just run:
  ```bash
  sudo apt install autogrid
  ```
* **UCSF Chimera / ChimeraX:** Used for molecular visualization, receptor cleaning, and selection of target residues.

---

## **2. Step-by-Step Workflow**

### **2.1: Receptor Preparation (General)**

1.  **Obtain Receptor Structure:** A PDB structure of the target receptor is required. For this workflow `5TBM` (HIF-2α PAS-B domain) was used as a base.
2.  **Clean Receptor:** Using chimera we removed the chain b, ligand and the solvent using: select then delete commands.
3.  **receptor preparation** In Chimera run Tools -> Structure Editing -> Dock Prep (add hydrogen atoms)
    * **Input file example:** `receptor_ready_5tbm.pdb` (This is the cleaned receptor PDB file).

### **2.2: Identifying and Selecting Target Residues (UCSF Chimera/ChimeraX)**

To focus the docking on a specific binding site within HIF-2α, key residues known or hypothesized to interact with inhibitors were selected gathered from the literature:

* **H293:** Direct Bond (H-bond), Allosteric Modulator
* **Y281:** Direct Bond (H-bond network, n → π\*Ar); Allosteric Modulator
* **M252:** Allosteric Modulator, Hydrophobic Pocket Constituent (initially)
* **S304:** Gating Residue, Pocket Architecture
* **G323:** Resistance Mutation Site, Pocket Lining
* **Y278:** Allosteric Modulator, Hydrophobic Pocket Constituent
* **L309:** Hydrophobic Pocket Constituent
* **V321:** Hydrophobic Pocket Constituent
* **F325:** Hydrophobic Pocket Constituent
* **N288:** Gating Residue
* **L272:** Gating Residue
* **A277:** (Primarily for agonist interaction, H-bond with backbone)
* **H248:** Pocket Boundary

**Action:**
Using UCSF Chimera or ChimeraX:
1.  The `receptor_ready_5tbm.pdb` was loaded.
2.  The residues listed above (or a relevant subset defining the entire target cavity) were selected.
3.  Only the atoms of these selected residues were saved to a new PDB file.
    * **Output custom PDB file example:** `custom_residues_5tbm_cleaned.pdb`

### **2.3: Generating a Focused Grid Parameter File (GPF) with Meeko**

Meeko's `mk_prepare_receptor.py` script was used to prepare the full receptor PDBQT and generate a GPF specifically targeted to the selected residues.

**Command Executed:**
```bash
mk_prepare_receptor.py --read_pdb receptor_ready_5tbm.pdb -o myreceptor_targeted -p -g --box_enveloping custom_residues_5tbm_cleaned.pdb --padding 5.0
```
#### **Explanation:**

* `--read_pdb receptor_ready_5tbm.pdb`: Specifies the input full receptor PDB.
* `-o myreceptor_targeted`: Sets the output basename for generated files.
* `-p`: Instructs Meeko to write the prepared receptor PDBQT file.
* `-g`: Instructs Meeko to write the Grid Parameter File (.gpf).
* `--box_enveloping custom_residues_5tbm_cleaned.pdb`: Crucially, uses the PDB file of selected residues to define the grid box center and base dimensions.
* `--padding 5.0`: Adds 5.0 Å padding around the selected residues.

#### **Output Files from this step:**

* `myreceptor_targeted.pdbqt`: The full receptor prepared in PDBQT format.
* `myreceptor_targeted.gpf`: The AutoGrid Parameter File, with grid dimensions centered on the selected residues.
* `myreceptor_targeted.box.pdb`: A PDB file representing the grid box for visualization.
* `boron-silicon-atom_par.dat`: Atomic parameters for B and Si.

### **2.5: Correcting the Grid Parameter File (GPF)**

An error was encountered when initially running `autogrid4`: `"autogrid4: ERROR: Too many "map" keywords (...); the "ligand_types" command declares only (...) atom types."` This indicated a mismatch between the number of atom types listed in the `ligand_types` line and the number of `map` lines in the `myreceptor_targeted.gpf` file, or that the total number of unique types exceeded AutoGrid's limit (often around 14 for atom-specific affinity maps).

**Original problematic `ligand_types` line (example from troubleshooting):**

Contained 17 type labels, with duplicates like `Cl` and `CL`)

**Correction Made:**
The `myreceptor_targeted.gpf` file was manually edited:

1.  The `ligand_types` line was corrected to remove duplicate/case-variant types (e.g., `CL` was removed, keeping `Cl`; `BR` was removed, keeping `Br`).
2.  To ensure the number of types was within AutoGrid's limit (~14 types, as you found worked), types like `Si` and `B` were also removed from the `ligand_types` line (assuming they are not essential for the specific receptor/ligands in this study).
* **final file can be found in the dictionary**
### **2.6: Then autogrid and docking commands:**
```bash
autogrid4 -p myreceptor_targeted.gpf -l myreceptor_targeted.glg
```
### **2.7: Convert smiles to PDBQT files**
1. First open the file **prepare_ligands.py**
change the line:
```python
input_smiles_file = "your_smiles_file.smi"  # <--- CHANGE THIS to process a different SMILES file
```
2. then, in the terminal, run:
```bash
prepare_ligands.py
```
3. if you wanna use parallized distrubtion for prepare ligands (useful if you have many molecuels to process), use:
```bash
   python prepare_ligands_parallel.py
 ```
* Remember to edit the file first and specify the number of threads you have, i have 32 so i used:
```python
   if __name__ == "__main__":
    prepare_ligands_from_smiles(
        smiles_file="undocked_smiles.smi",
        out_dir="ligands_pdbqt_part1",
        n_workers=32,          # ← change this to number of cores you have 
    )
```
### **2.8: Run docking**
* First make sure you have the right setup:
```shell
 === Configuration ===
# 1. FULL PATH to your compiled AutoDock-GPU executable
AUTODOCK_GPU_EXEC="/home/joe/projects/AutoDOCK/AutoDock-GPU/bin/autodock_gpu_128wi"

# 2. Name of your PRE-CALCULATED receptor grid map file (.maps.fld)
MAP_FILE_FLD_FILENAME="myreceptor_targeted.maps.fld"

# 3. Name of the directory containing your ligand PDBQT files
LIGAND_DIR="ligands_pdbqt"

# 4. Name of the directory where docking results (DLG files) will be saved
OUTPUT_DIR="docking_results_compiled"

# 5. Docking run parameters
NUM_RUNS="10"              # Number of docking runs per ligand
DEVICE_NUMBER="2"          # SET THIS to the GPU device number (e.g., 1 for the first GPU, 2 for the second) only if you use run_docking_with_gpu.sh
```
* Then run:

```bash
bash run_docking_batch.sh
```
* Or with gpu selection:
```bash
bash run_docking_with_gpu.sh
```

### **2.9: the output file was converted to pdbqt using:**
```bash
chmod +x convert_dlgs_to_pdbqt.sh
./convert_dlgs_to_pdbqt.sh
```
* it requires MGLToolsPckgs, to install run either:
```bash
  conda install bioconda::mgltools
   conda install bioconda/label/cf201901::mgltools
```
### **2.10: filter the ligands that have docked outside the pocket:**
with **what** environment active run:
```bash
python filter_docked_ligands.py
```
### **2.11: rank the files by their energy and copy the top 100 compound into a separate folder:**
with **what** environment active run:
```bash
python rank_docking_results.py
```
the following settings applied:
```python
INPUT_DLG_DIR   = Path("docking_results_compiled")
INPUT_PDBQT_DIR = Path("docking_converted_filtered")
TOP_OUT_DIR     = Path("top_100_ligands_simple")
CSV_OUT         = Path("docking_results_ranked_simple.csv")
SMILES_INPUT_FILE = Path("50k.smi") # Added SMILES input file
TOP_N           = 100
```
---

## 3: **Isolate undocked compounds**
To isolate undocked compounds, open separate_undocked.py and edit:

```python
csv_file_path = 'results/docking_results_ranked_docked.csv' # Path to the CSV file containing docking results
smiles_file_path = '1m.smi' # Path to the SMILES file containing all compounds
new_smiles_file_path = 'undocked_smiles.smi'    # Path to save the undocked SMILES
```
then run
```bash
python seprate_undocked.py
```
To separate batches for docking, run:
```
python give_me_n_compounds.py undocked_smiles.smi 100000 --parts 2 --output_prefix undocked_100K
```
You can change 100000 to number fo compounds you want and 2 to number of batches (parts you want) and undocked_100K to the prefix of the compound 

**Youssef Abo-Dahab, Pharm.D**  
*M.S. Candidate, AI & Computational Drug Discovery*  
*Bioengineering and Therapeutic Sciences Department*  
University of California, San Francisco (UCSF)  
 📧 youssef.abo-dahab@ucsf.edu
