[//]: # (Author: Lachlan Whyborn)
[//]: # (Date Modified: )

# Running CABLE Configurations in Payu

This repository serves as a template for multi-stage CABLE configurations both through (Payu)[https://github.com/payu-org/payu] and through a stand-alone CABLE runner (to come).

## Usage

The payu configuration in *config.yaml* remains identical (see the full documentation on payu configurations (here)[https://payu.readthedocs.io/en/latest/config.html]). The additions for CABLE configurations are the *stage_config.yaml* file and the stage namelist directories. The ```stage_config.yaml``` prescribes the stages (and how many times they run), that form the CABLE configuration in order and the locations of the namelists which define the actions of each stage. The branches of this repository contain example configurations.

Once the laboratory is configured, invoke ```payu run``` to execute the CABLE configuration. Note that if CABLE exits with a non-success exit code, invoking ```payu run``` again will by default restart from the stage that failed (after a ```payu sweep```). To begin from the start, remove the ```configuration_log.yaml``` file with ```rm configuration_log.yaml```.

### Modifying Namelists

Namelists for each stage are modified by having "patch" namelists for each stage. A directory in the laboratory for each stage specified in ```stage_config.yaml``` is required, which contains the patch namelists that will be applied to the top level namelist before running the stage e.g. when running a stage named ```climate_spinup```, a directory named ```climate_spinup``` must exist, and it *may* contain ```cable.nml```, ```luc.nml``` and ```cru.nml``` (note that it doesn't need to contain all the namelists). These namelists are patched into to the respective top level namelists. Options specified in the stage namelist take precedence. If the directory does not contain a given namelist, the top level namelist will be taken as is.

### Accessing Restarts

The point of these multi-stage configurations is to spin-up various physical processes to a quasi-equilibrium, for use in later simulations. As such, we want to capture outputs or restarts from a previous stage and use them as inputs to the next stage. This is achieved by symlinking the contents of the ```restart``` directories from the previous stages to the working directory for the current stage. The most recent version of a given restart file will be taken in the instance of multiple stages generating the same file.

### Configuration Progress

The progress of the configuration is monitored by the ```configuration_log.yaml``` file, which lists the queued and completed stages, and the stage that is currently running. Once a stage is finished running, it is stored in ```archive/output{id:%03d}/```, where ```id``` is the number of the current stage (starting at 0). A copy of the configuration log, snapshotted at the point the stage began, is stored in the output directory to assist interpretation of results.
