# System preparation
This folder stores the input and output files of each step for preparing the NaCl system. The initial files for the protocol include the following:
- `sys_init.gro`
- `sys.top` (in the folder `Topology`, later renamed as `#sys.top.1#`) 
- The folder `mdp_files` that contains `.mdp` files for different steps, including
  - Energy minimization (`em.mdp`)
  - NVT equilibration (`nvt_equil.mdp`)
  - NPT equilibration (`npt_equil.mdp`) 
  - NPT MD simulation (`md.mdp`)

For convenience, a folder was created for each of the steps that required an `.mdp` file and the corresponding `.mdp` was copied over. With these files, here we summarize the protocol for preparing the NaCl system for the tutorials of advanced sampling methods. Note that here we will only brief describe the commands to use as we assume that the reader is familiar with basics of MD simulations. If not, consider reading Lemkul's tutorials on LiveCoMS[^1] up to section 3.2. 

## 1. Solvation
First we place NaCl in the center of a cubic box (`-c`), with 1 nm between the solute and box edges (`-d 1`). This can be done by executing the following command (working directory: `./`):
```
gmx editconf -f sys_init.gro -o sys_box.gro -c -d 1
```
Then, we add solvent to the simulation box (working directory: `./`). 
```
gmx solvate -cp sys_box.gro -p Topology/sys.top -o sys_sol.gro
```

## 2. Energy minimization (EM)
After assembling NaCl with the solvent molecules, we need to energy-minimize the system to ensure there are no steric clashes. To do this, we first need to generate a binary input file (`.tpr`) using the `grompp` command, which reads in the system coordinates (`sys_sol.gro`), topology (`Topology/sys.top`) and parameters for EM (`em.mdp`). The working directory is `EM/`.
```
gmx grompp -f em.mdp -c ../sys_sol.gro -p ../Topology/sys.top -o em.tpr
```
Then, we use `mdrun` to perform energy minimization (working directory : `EM/`):
```
gmx mdrun -s em.tpr -deffnm em
```

## 3. NVT equilibration
Similarly, we first use `grompp` to generate a `.tpr` file (working directory: `Equil/NVT/`):
```
gmx grompp -f nvt_equil.mdp -c ../../EM/em.gro -p ../../Topology/sys.top -o nvt_equil.tpr
```
Then, we perform NVT equilibration (working directory: `Equil/NVT/`):
```
gmx mdrun -s nvt_equil.tpr -deffnm nvt_equil
```

## 4. NPT equilibration
First, we generate a `.tpr` file  (working directory: `Equil/NPT/`):
```
gmx grompp -f npt_equil.mdp -c ../NVT/nvt_equil.gro -t ../NVT/nvt_equil.cpt -p ../../Topology/sys.top -o npt_equil.tpr -maxwarn 1
```
Then, we performed NPT equilibration (working directory: `Equil/NPT/`)
```
gmx mdrun -s npt_equil.tpr -deffnm npt_equil
```

## 5. MD simulation
Again, we generate a `.tpr` file using `grompp` (working directory: `MD/`):
```
gmx grompp -f md.mdp -c ../Equil/NPT/npt_equil.gro -p ../Topology/sys.top -t ../Equil/NPT/npt_equil.cpt -o md.tpr
```
Then, we use the `mdrun` command to run a standard MD simulation in the NPT ensemble. 
```
mdrun -s md.tpr -deffnm md
```
After the simulation is done, we can create a folder `Final_inputs` and copy over the output configuration of the short MD simulation and the system topology. By putting the simulation inputs in the same folder, it is easier to make sure that the same set of input files are used in all advanced sampling simulations we are going to run later in the tutorial.
```
# Working directory: ./
mkdir Final_inputs && cd Final_inputs
mv ../MD/md.gro sys.gro
mv ../Topology/sys.top . 
```
### 6. Analysis of the CV timeseries
Before embarking on metadynamics simulations in our tutorial, a preliminary analysis of the CV time series can yield valuable insights. One important consideration is the appropriate Gaussian width to use in the metadynamics simulation, which can be determined by calculating the standard deviation of the CV from an unbiased/standard molecular dynamics (MD) simulation. To this end, our goal is to determine the standard deviation of the ion-pair distance from the time series generated from the completed MD simulation. Such a post-simulation analysis can be done by the `plumed driver` tool in PLUMED, given a simulation trajectory. `plumed driver` requires a PLUMED input file, which in our case has the following content:
```
d: DISTANCE ATOMS=1,2
PRINT ARG=d, STRIDE=1 FILE=dist.dat
```
With the plumed.dat file in place, we can then run the plumed driver by executing the following command from the MD/ working directory:
```
plumed driver --mf_xtc md.xtc --plumed plumed.dat --timestep 20
```
Upon completion, the output file dist.dat will be generated, which contains the time series of the ion-pair distance. It is important to note that the time step in this series will match the stride used to update the .xtc file during the simulation (specified by the nstxout-compressed parameter), which was set to 10000 simulation steps or 20 ps in our case. Thus, a value of 20 was specified for the --timestep flag to ensure that the time frames in the output file are correct.

If a higher time-resolution of the time series is desired, one could consider 
- Increasing the output frequency of the `.xtc` file, though this will significantly increase disk space usage.
- Monitoring the CV on-the-fly during the simulation, which requires `mdrun` to additionally read in a PLUMED input file like the one we used above. This is how `dist.dat` in the `MD` folder here was generated.

In the second case, the time step in the output time series will be equal to the integration step (0.002 ps) multiplied by the value of the `STRIDE` parameter in the PLUMED input file. As can be checked, the PLUMED input file `plumed.dat` used with MD simulation in our case used `STRIDE=100`, resulting a time step of 0.2 ps in the output time series, a much higher resolution compared to using `plumed driver` post-simulation. With `plumed.dat`, one can monitor a CV when running the simulation by using the following command:
```
gmx mdrun -s md.tpr -deffnm md -plumed plumed.dat
```
Once the simulation is completed, `dist.dat` will be generated, from which we can calculate the ion-pair distance using the following Pytho ncode:
```python
import plumed
import numpy as np
data = plumed.read_as_pandas('dist.dat')
print(f"{np.mean(data['d']): .3f} +/- {np.std(data['d']): .3f}")
```
As a result, the ion-pair distance during the simulation was around 1.002 +/- 0.358 nm, and 0.358 can therefore be used as a reasonable Gaussian width in metadynamics simulations biasing the ion-pair distance.

[^1]: Lemkul. (2018) "From Proteins to Perturbed Hamiltonians: A Suite of Tutorials for the GROMACS-2018 Molecular Simulation Package." Living J. Comp. Mol. Sci, 1, 5068.
