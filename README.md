cs598-dl4h-final-project-team120

### Folder Specification
- data
    - **processing.py**: our data preprocessing file.
    - Input (extracted from external resources)
        - PRESCRIPTIONS.csv: the prescription file from MIMIC-III raw dataset
        - DIAGNOSES_ICD.csv: the diagnosis file from MIMIC-III raw dataset
        - PROCEDURES_ICD.csv: the procedure file from MIMIC-III raw dataset
        - RXCUI2atc4.csv: this is a NDC-RXCUI-ATC4 mapping file, and we only need the RXCUI to ATC4 mapping. This file is obtained from https://github.com/sjy1203/GAMENet, where the name is called ndc2atc_level4.csv.
        - drug-atc.csv: this is a CID-ATC file, which gives the mapping from CID code to detailed ATC code (we will use the prefix of the ATC code latter for aggregation). This file is obtained from https://github.com/sjy1203/GAMENet.
        - rxnorm2RXCUI.txt: rxnorm to RXCUI mapping file. This file is obtained from https://github.com/sjy1203/GAMENet, where the name is called ndc2rxnorm_mapping.csv.
        - drugbank_drugs_info.csv: drug information table downloaded from drugbank here https://www.dropbox.com/s/angoirabxurjljh/drugbank_drugs_info.csv?dl=0, which is used to map drug name to drug SMILES string.
        - drug-DDI.csv: this a large file, containing the drug DDI information, coded by CID. The file could be downloaded from https://drive.google.com/file/d/1mnPc0O0ztz0fkv3HF-dpmBb8PLWsEoDz/view?usp=sharing
    - Output
        - atc3toSMILES.pkl: drug ID (we use ATC-3 level code to represent drug ID) to drug SMILES string dict
        - ddi_A_final.pkl: ddi adjacency matrix
        - ddi_matrix_H.pkl: H mask structure (This file is created by **ddi_mask_H.py**)
        - ehr_adj_final.pkl: used in GAMENet baseline (if two drugs appear in one set, then they are connected)
        - records_final.pkl: The final diagnosis-procedure-medication EHR records of each patient, used for train/val/test split.
        - voc_final.pkl: diag/prod/med index to code dictionary
- src/
    - SafeDrug.py: our model
    - baselines:
        - GAMENet.py
        - Leap.py
        - Retain.py
        - ECC.py
        - LR.py
    - setting file
        - model.py
        - util.py
        - layer.py

> The current statistics are shown below:
```
#patients  6350
#clinical events  15032
#diagnosis  1958
#med  112
#procedure 1430
#avg of diagnoses  10.5089143161256
#avg of medicines  11.647751463544438
#avg of procedures  3.8436668440659925
#avg of vists  2.367244094488189
#max of diagnoses  128
#max of medicines  64
#max of procedures  50
#max of visit  29
```


### Step 1: Package Dependency

- first, install the rdkit conda environment
```python
conda create -c conda-forge -n SafeDrug  rdkit
conda activate SafeDrug
```

- then, in SafeDrug environment, install the following package
```python
pip install scikit-learn, dill
```
Note that torch setup may vary according to GPU hardware. Generally, run the following
```python
pip install torch

- Finally, install other packages if necessary
```python
pip install [xxx] # any required package if necessary, maybe do not specify the version, the packages should be compatible with rdkit
```

Here is a list of reference versions for all package

```shell
Python: 3.8.12
pandas: 1.4.1
dill: 0.3.4
torch: 1.8.0
rdkit: 2022.03.1
scikit-learn: 1.0.2
numpy: 1.19.5
```

Let us know any of the package dependency issue. Please pay special attention to pandas, some report that a high version of pandas would raise error for dill loading.


### Step 2: Data Processing

- Go to https://physionet.org/content/mimiciii/1.4/ to download the MIMIC-III dataset (You may need to get the certificate)

  ```python
  cd ./data
  wget -r -N -c -np --user [account] --ask-password https://physionet.org/files/mimiciii/1.4/
  ```

- go into the folder and unzip three main files

  ```python
  cd ./physionet.org/files/mimiciii/1.4
  gzip -d PROCEDURES_ICD.csv.gz # procedure information
  gzip -d PRESCRIPTIONS.csv.gz  # prescription information
  gzip -d DIAGNOSES_ICD.csv.gz  # diagnosis information
  ```

- download the DDI file and move it to the data folder
  download https://drive.google.com/file/d/1mnPc0O0ztz0fkv3HF-dpmBb8PLWsEoDz/view?usp=sharing
  ```python
  mv drug-DDI.csv ./data
  ```

- processing the data to get a complete records_final.pkl

  ```python
  cd ./data
  vim processing.py
  
  # line 323-325
  # med_file = './physionet.org/files/mimiciii/1.4/PRESCRIPTIONS.csv'
  # diag_file = './physionet.org/files/mimiciii/1.4/DIAGNOSES_ICD.csv'
  # procedure_file = './physionet.org/files/mimiciii/1.4/PROCEDURES_ICD.csv'
  
  python processing.py
  
  # please change into your own MIMIC folder
  
    med_file = './input/PRESCRIPTIONS.csv'
    diag_file = './input/DIAGNOSES_ICD.csv'
    procedure_file = './input/PROCEDURES_ICD.csv'
	
    # input auxiliary files
    med_structure_file = './output/atc32SMILES.pkl'
    RXCUI2atc4_file = './input/RXCUI2atc4.csv' 
    cid2atc6_file = './input/drug-atc.csv'
    rxnorm2RXCUI_file = './input/rxnorm2RXCUI.txt'
    ddi_file = './input/drug-DDI.csv'
    drugbankinfo = './input/drugbank_drugs_info.csv'

    # output files
    ddi_adjacency_file = "./output/ddi_A_final.pkl"
    ehr_adjacency_file = "./output/ehr_adj_final.pkl"
    ehr_sequence_file = "./output/records_final.pkl"
    vocabulary_file = "./output/voc_final.pkl"
    ddi_mask_H_file = "./output/ddi_mask_H.pkl"
    atc3toSMILES_file = './output/atc3toSMILES.pkl’
  ```


### Step 3: run the code

```python
Notes: 
Set device='cpu' to run code on a CPU based environment otherwise you may get CUDA assertion error required for GPU environment
Update input/output data/model output file paths to your local environment

python SafeDrug.py
```

here is the argument:

    usage: SafeDrug.py [-h] [--Test] [--model_name MODEL_NAME]
                   [--resume_path RESUME_PATH] [--lr LR]
                   [--target_ddi TARGET_DDI] [--kp KP] [--dim DIM]
    
    optional arguments:
      -h, --help            show this help message and exit
      --Test                test mode
      --model_name MODEL_NAME
                            model name
      --resume_path RESUME_PATH
                            resume path
      --lr LR               learning rate
      --target_ddi TARGET_DDI
                            target ddi
      --kp KP               coefficient of P signal
      --dim DIM             dimension


Machine 1 Details: 8 GB RAM, x64 4 Core AMD Ryzen 5 PRO Processor [Windows]
Machine 2 Details: 32 GB RAM, Mac M1 Pro Processor
Note Machine 2 was used for obtaining all results with all baselines and the SafeDrug model.

### Citation
```bibtex
@inproceedings{yang2021safedrug,
    title = {SafeDrug: Dual Molecular Graph Encoders for Safe Drug Recommendations},
    author = {Yang, Chaoqi and Xiao, Cao and Ma, Fenglong and Glass, Lucas and Sun, Jimeng},
    booktitle = {Proceedings of the Thirtieth International Joint Conference on
               Artificial Intelligence, {IJCAI} 2021},
    year = {2021}
}
```

Welcome to contact me <chaoqiy2@illinois.edu> for any question. Partial credit to a previous repo (this paper is also from our group) https://github.com/sjy1203/GAMENet
