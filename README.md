# DUNE_PRISM_GEC_ND
DUNE-PRISM: GEC from muon side
> - ND CAF Maker: https://internal.dunescience.org/doxygen/ND__CAFMaker_2dumpTree_8py_source.html
> - Some are copied from: [FNAL instruction](https://github.com/FlynnYGUO/NeutrinoPhysics/blob/main/GEC/BaronCodeOutdated/NDGEC.md) and [NNhome instruction](https://github.com/FlynnYGUO/NeutrinoPhysics/blob/main/GEC/BaronNewCode/Instructions.md)
> [DUNE Computing Training](https://dune.github.io/computing-basics/index.html)
## FNAL machine
### 0. Setup
#### 1. Log in & DUNE FNAL machines (dunegpvm*) environment setup:
```
kfnal                                      # Short for kinit -f <username>@FNAL.GOV. In my laptop, alias kfnal="/usr/bin/kinit flynnguo@FNAL.GOV" in ~/.zshrc
ssh -X flynnguo@dunegpvm15.fnal.gov
exit                                       # Quit FNAL
```
Environment setup (only do it once):
```
cd /dune/app/users/flynnguo                                             # Replace with your username for all commands below
git clone https://github.com/FlynnYGUO/DUNE_PRISM_GEC_ND.git
# This allows using pip
source /cvmfs/dune.opensciencegrid.org/products/dune/setup_dune.sh
setup dunetpc v09_41_00_02 -q e20:prof
pip install --target=/dune/app/users/flynnguo/lib/python3.9/site-packages uproot
pip install --target=/dune/app/users/flynnguo/lib/python3.9/site-packages torch
pip install --target=/dune/app/users/flynnguo/lib/python3.9/site-packages scipy
```
Next time once you log in to the FNAL machine, do the following (do it every time you log in):
```
cd /dune/app/users/flynnguo
source /cvmfs/dune.opensciencegrid.org/products/dune/setup_dune.sh
setup dunetpc v09_41_00_02 -q e20:prof
export PYTHONPATH=/dune/app/users/flynnguo/lib/python3.9/site-packages:$PYTHONPATH
```
It would be better if you can source the new version of ROOT:
```
source /cvmfs/sft.cern.ch/lcg/app/releases/ROOT/6.24.08/x86_64-centos7-gcc48-opt/bin/thisroot.sh
```
Files I/O on DUNE machines
Input ND CAF files are here: ```/pnfs/dune/persistent/physicsgroups/dunelbl/abooth/PRISM/Production/Simulation/ND_CAFMaker/v7/CAF```
Output files from grid jobs are written to the scratch area ```/pnfs/dune/scratch/users/<your username>```.

Please avoid reading from, copying from, or writing massive amount of files directly to the pnfs area ```/pnfs/dune/persistent```, this will slow down the file system. Refer to [this wiki](https://mu2ewiki.fnal.gov/wiki/DataTransfer) for good practices on data transfer.

#### 2. Interactive run
If you want to run code interactively on ```dunegpvm*``` for debugging, follow instruction in this section.
```
cd DUNE_PRISM_GEC_ND/code
python3 new_hadron_muon_mktree.py /pnfs/dune/persistent/physicsgroups/dunelbl/abooth/PRISM/Production/Simulation/ND_CAFMaker/v7/CAF/0mgsimple/101/FHC.1101999.CAF.root
```

#### 3.1 Submit grid job (Geometric efficiency for ND events at ND)
```
# Make a tarball to send everything you need to run your program on grid node
cd DUNE_PRISM_GEC_ND/code
# Write list of files into a txt file, remember to change the foldername before submitting job
ls -d "/pnfs/dune/persistent/physicsgroups/dunelbl/abooth/PRISM/Production/Simulation/ND_CAFMaker/v7/CAF/<folder_name>”/* | sed "s\/pnfs\root://fndca1.fnal.gov:1094/pnfs/fnal.gov/usr\g" > NDCAFs.txt
# For example:
ls -d "/pnfs/dune/persistent/physicsgroups/dunelbl/abooth/PRISM/Production/Simulation/ND_CAFMaker/v7/CAF/0mgsimple/"{110,111,112,113,114,115,116,117,118,119}/* | sed "s\/pnfs\root://fndca1.fnal.gov:1094/pnfs/fnal.gov/usr\g" > NDCAFs.txt
# Now make a tarball
tar -czvf ND_work.tar.gz setup_NDcombEff.sh new_hadron_muon_mktree.py muonEff30.nn muonEffModel.py NDCAFs.txt


# The following long command submits your job:
# -N 2 means 2 jobs, this is now set as running 1 file per job as I have two files in txt file.
# If you have X files in your text file, set: -N X
# You can use ```wc -l NDCAFs.txt``` to check the number of files in txt file
jobsub_submit -G dune -N 9980 --memory=5GB --disk=10GB --expected-lifetime=1.5h --cpu=1 --resource-provides=usage_model=DEDICATED,OPPORTUNISTIC,OFFSITE --tar_file_name=dropbox:///dune/app/users/flynnguo/DUNE_PRISM_GEC_ND/code/work.tar.gz --use-cvmfs-dropbox -l '+SingularityImage=\"/cvmfs/singularity.opensciencegrid.org/fermilab/fnal-wn-sl7:latest\"' --append_condor_requirements='(TARGET.HAS_Singularity==true&&TARGET.HAS_CVMFS_dune_opensciencegrid_org==true&&TARGET.HAS_CVMFS_larsoft_opensciencegrid_org==true&&TARGET.CVMFS_dune_opensciencegrid_org_REVISION>=1105&&TARGET.HAS_CVMFS_fifeuser1_opensciencegrid_org==true&&TARGET.HAS_CVMFS_fifeuser2_opensciencegrid_org==true&&TARGET.HAS_CVMFS_fifeuser3_opensciencegrid_org==true&&TARGET.HAS_CVMFS_fifeuser4_opensciencegrid_org==true)' file:///dune/app/users/flynnguo/DUNE_PRISM_GEC_ND/code/run_NDcombEff.sh
```

To query the job status: ```jobsub_q <usrname> -G dune```. [See job details](https://fifemon.fnal.gov/monitor/d/000000115/job-cluster-summary?orgId=1&var-cluster=73871417&var-schedd=jobsub02.fnal.gov)  
To remove a job status: ```condor_rm <usrname> -G dune```

If your job gets held (HoldReasonCode 26), subcode 1 is memory, 2 is disk, 4 is too many starts (max of 10 attempts to run a job), 8 is run time. You can can also get HoldReasonCode 34 and subcode 0, which is also memory.

The job script (```run_NDcombEff.sh```) can be adjusted to run more than one file per job, if desired, contact me. The magic happens at these lines:
```
(( LINE_N = ${PROCESS} + 1 ))

for ifile in $(cat ${INPUT_TAR_DIR_LOCAL}/NDCAFs.txt | head -${LINE_N} | tail -1); do
```
#### 3.2 Submit grid job (Geometric efficiency for FD events at ND)
```
# Make a tarball to send everything you need to run your program on grid node
cd DUNE_PRISM_GEC_ND/code
# Write list of files into a txt file, remember to change the foldername before submitting job
ls -d "/pnfs/dune/persistent/users/flynnguo/FDGeoEffinND/<FDGeoEff_JOBID>"/* | sed "s\/pnfs\root://fndca1.fnal.gov:1094/pnfs/fnal.gov/usr\g" > FDCAFs.txt
# For example:
ls -d "/pnfs/dune/persistent/users/flynnguo/FDGeoEffinND/FDGeoEff_2811722"/* | sed "s\/pnfs\root://fndca1.fnal.gov:1094/pnfs/fnal.gov/usr\g" > FDCAFs.txt
# Now make a tarball
tar -czvf FD_work.tar.gz setup_NDcombEff.sh FD_maketree.py muonEff30.nn muonEffModel.py FDCAFs.txt
# You can use ```wc -l FDCAFs.txt``` to check the number of files in txt file
jobsub_submit -G dune -N 8057 --memory=5GB --disk=10GB --expected-lifetime=8h --cpu=1 --resource-provides=usage_model=DEDICATED,OPPORTUNISTIC,OFFSITE --tar_file_name=dropbox:///dune/app/users/flynnguo/DUNE_PRISM_GEC_ND/code/FD_work.tar.gz --use-cvmfs-dropbox -l '+SingularityImage=\"/cvmfs/singularity.opensciencegrid.org/fermilab/fnal-wn-sl7:latest\"' --append_condor_requirements='(TARGET.HAS_Singularity==true&&TARGET.HAS_CVMFS_dune_opensciencegrid_org==true&&TARGET.HAS_CVMFS_larsoft_opensciencegrid_org==true&&TARGET.CVMFS_dune_opensciencegrid_org_REVISION>=1105&&TARGET.HAS_CVMFS_fifeuser1_opensciencegrid_org==true&&TARGET.HAS_CVMFS_fifeuser2_opensciencegrid_org==true&&TARGET.HAS_CVMFS_fifeuser3_opensciencegrid_org==true&&TARGET.HAS_CVMFS_fifeuser4_opensciencegrid_org==true)' file:///dune/app/users/flynnguo/DUNE_PRISM_GEC_ND/code/run_FDcombEff.sh
```

## NNhome machine
### 0. Setup
#### 1. Log in:
```
ssh -X fyguo@nnhome.physics.sunysb.edu       # Log my ivy account: <username>@ivy.physics.sunysb.edu
passwd                                     # Reset my password  
exit                                       # Quit ivy
```
#### 2. Install packages/tools
```
pip install --target=<a directory you specify> uproot  # Install uproot, you may also need to install torch
source /home/rrazakami/workspace/ROOT/root_binary/bin/thisroot.sh  # Use other's ROOT source file instead of installing a new one under my repository
# Remember to source root.sh every time once log in the NNhome machine
```
#### 3. Muon NN
The network outputs how probable a muon is fully contained in ND LAr and tracker matched in TMS downstream: https://github.com/weishi10141993/MuonEffNN
The current used network file is located at ```/home/barwu/repos/MuonEffNN/8thTry/muonEff30.nn```

### I. Get ND eff files
The ND CAFs have been copied from Fermilab to NNhome under this path:
```
/storage/shared/barwu/10thTry/NDCAF
```
They are grouped into different folders, as well as numerical subfolders. You will need to check these names to decide which of the subfolders you will run.

The following scripts are used to produce raw, selected, and efficiency-corrected distributions. These are the red-green-blue curves for muon only (fully contained and tracker matched), hadron only, and combined.

Run the script using python3, this creates a TTree containing all efficiency information:
```
python3 /home/fyguo/DUNE_PRISM_GEC_ND/code/new_hadron_muon_mktree.py [folder]/[subfolder]
```
A batch submission is also available, it usually takes half a day to complete:
```
sbatch Slurm_nnhome_ND.sh [folder]/[subfolder] # For example, Omgsimple/104
# There are four GPU node lists: fir, birch, aspen, cedar
```
The batch job should generate 1000 efficiency files. These files contain the efficiency information of each event, the cut information, as well as 3 additional sets of information about the total lepton momentum, longitudinal lepton momentum and the cosine of the angle between the neutrino and lepton momentum vectors.

To convert all the efficiency files into a set of histograms, run:
```
root -l -b /home/fyguo/DUNE_PRISM_GEC_ND/code/histogram_files_ND.cpp
```
They have been organized depending on what parameter is being plotted, for example, neutrino energy or position. In each group, it contains graphs for raw, selection-cut and
geometrically-corrected distributions.

Now run the script to draw the histograms using the aforementioned TFiles:
```
root -l -b
.L /home/fyguo/DUNE_PRISM_GEC_ND/code/draw_histograms_ND.cpp
draw_histograms_ND(<geoeff_cut>) #For example: draw_histograms_ND(0.1) means 10% geoeff_cut
```
There will be a set of all graphs available, and a copy of each of those graphs organized by selection-cut into different TCanvases. Ratio plots for each graph are also available. The graphs and ratio plots will automatically save once they are finished loading.

### II. Get FD eff files
Run this script:
```
python3 /home/fyguo/DUNE_PRISM_GEC_ND/code/FD_maketree.py
```
This generates efficiency files for far detector events.

To submit a job:
```
sbatch Slurm_nnhome_FD.sh
```

Run this script to draw histograms directly using the FD CAFs and efficiency files produced by the above shell script:
```
root -l -b
.L /home/fyguo/DUNE_PRISM_GEC_ND/code/draw_histograms_FD.cpp
draw_histograms_FD(<geoeff_cut>) #For example: draw_histograms_FD(0.1) means 10% geoeff_cut
```

### III. Get ND ratios vs FD ratios
Generate all ND ratios, FD ratios and NDvsFD ratios.
```
root -l -b /home/fyguo/DUNE_PRISM_GEC_ND/code/NDaFD_RatioPlots.cpp
```
