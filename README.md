[//]: # (Author: Lachlan Whyborn)
[//]: # (Date Modified: )

# Running CABLE Configurations in Payu

This repository serves as a template for multi-stage CABLE configurations both through (Payu)[https://github.com/payu-org/payu] and through a stand-alone CABLE runner (to come).

## Usage

The payu configuration in *config.yaml* remains identical (see the full documentation on payu configurations (here)[https://payu.readthedocs.io/en/latest/config.html]). The additions for CABLE configurations are the *stage_config.yaml* file and the stage namelist directories. The ```stage_config.yaml``` prescribes the stages (and how many times they run), that form the CABLE configuration in order and the locations of the namelists which define the actions of each stage.

As with all payu laboratories, the namelists required for the relevant version of CABLE must be included in the experiment directory. This repository was built targeting the (CABLE-POP\_TRENDY)[https://github.com/CABLE-LSM/CABLE/tree/CABLE-POP\_TRENDY] branch of the code, which requires the namelists ```cable.nml```, ```luc.nml```, ```met_names.nml``` and a namelist for the meteorological forcing, in this case ```cru.nml```.

Once the laboratory is configured, invoke ```payu run``` to execute the CABLE configuration. Note that if CABLE exits with a non-success exit code, invoking ```payu run``` again will by default restart from the stage that failed (after a ```payu sweep```). To begin from the start, remove the ```configuration_log.yaml``` file with ```rm configuration_log.yaml```.

### Modifying Namelists

Namelists for each stage are modified by having "patch" namelists for each stage. A directory in the laboratory for each stage specified in ```stage_config.yaml``` is required, which contains the patch namelists that will be applied to the top level namelist before running the stage e.g. when running a stage named ```climate_spinup```, a directory named ```climate_spinup``` must exist, and it *may* contain ```cable.nml```, ```luc.nml``` and ```cru.nml``` (note that it doesn't need to contain all the namelists). These namelists are patched into to the respective top level namelists. Options specified in the stage namelist take precedence. If the directory does not contain a given namelist, the top level namelist will be taken as is.

### Accessing Restarts

The point of these multi-stage configurations is to spin-up various physical processes to a quasi-equilibrium, for use in later simulations. As such, we want to capture outputs or restarts from a previous stage and use them as inputs to the next stage. This is achieved by symlinking the contents of the ```restart``` directories from the previous stages to the working directory for the current stage. The most recent version of a given restart file will be taken in the instance of multiple stages generating the same file.

### Configuration Progress

The progress of the configuration is monitored by the ```configuration_log.yaml``` file, which lists the queued and completed stages, and the stage that is currently running. Once a stage is finished running, it is stored in ```archive/output{id:%03d}/```, where ```id``` is the number of the current stage (starting at 0). A copy of the configuration log, snapshotted at the point the stage began, is stored in the output directory to assist interpretation of results.

## A Simple Example

Here is an example of a simple configuration which contains 3 spin-up stages, one each for the climate, biomass and land use change. The ```stage_config.yaml``` for such a configuration would look like:

*stage_config.yaml*
```
climate_spinup:
    count: 1
biomass_spinup:
    count: 1
land_use_spinup:
    count: 1
```

The order in which the stages appear in ```stage_config.yaml``` is the order in which the stages are executed. The climate spin-up stage requires namelist modifications to ```cable.nml```, the biomass spin-up requires namelist modifications to ```cable.nml``` and ```cru.nml``` and the land use change spin-up requires changes to all three namelists. The directory structure of the laboratory would then be:

```
[Staged-CABLE-in-Payu]$ git checkout simple-3stage-example
[Staged-CABLE-in-Payu]$ tree --dirsfirst
.
├── biomass_spinup
│   ├── cable.nml
│   └── cru.nml
├── climate_spinup
│   └── cable.nml
├── land_use_spinup
│   ├── cable.nml
│   ├── cru.nml
│   └── luc.nml
├── cable.nml
├── config.yaml
├── cru.nml
├── luc.nml
├── met_names.nml
└── stage_config.yaml
```

In this example, the climate spin-up stage provides CABLE and climate restarts (```filename%restart_in``` and ```cable_user%restart_in``` in ```cable.nml```) to the biomass spin-up, and the biomass spin-up provides CABLE, climate and CASA restarts (```casafile%ncpipool``` in ```cable.nml```). The stage namelists required to achieve this would be:

*climate_spinup/cable.nml*
```
&cablenml
    filename%restart_out = "restart/cable_rst.nc"
    cable_user%climate_restart_out = "restart/climate_rst.nc"
    ...
/
```

*biomass_spinup/cable.nml*
```
&cablenml
    filename%restart_in = "cable_rst.nc"
    filename%restart_out = "restart/cable_rst.nc"
    cable_user%climate_restart_in = "climate_rst.nc"
    cable_user%climate_restart_out = "restart/climate_rst.nc"
    casafile%cnpepool = "restart/CASA_rst.nc"
    ...
/

*land_use_spinup/cable.nml*
```
    filename%restart_in = "cable_rst.nc"
    cable_user%climate_restart_in = "climate_rst.nc"
    casafile%cnpipool = "CASA_rst.nc"
    ...
/
```

## An Advanced Example- the TRENDY Configuration

The TRENDY experiments are currently the most advanced configuation run in CABLE. It contains multi-step spin-up stages and retrieving restart files from earlier versions of that given restart. The ```stage_config.yaml``` file for this configuration is:

*stage_config.yaml*
```
climate_spinup:
    count: 1
biomass_spinup:
    count: 1
multistep_unrestricted_N_P:
    unrestricted_N_P:
        count: 2
    unrestricted_N_P_analytic:
        count: 2
multistep_restricted_N_P:
    restricted_N_P:
        count: 5
    restricted_N_P_analytic:
        count: 3
land_use_spinup:
    count: 1
full_dynamic_spinup:
    count: 1
```

Multi-step stages, denoted by the prefix ```multistep``` are stages which contain sub-steps which are cycled through ```count``` times. In this instance, the ```multistep_unrestricted_N_P``` stage is expanded as ```[unrestricted_N_P, unrestricted_N_P_analytic, unrestricted_N_P, unrestricted_N_P_analytic]```, as each internal step has a count of 2. The ```multistep_restricted_N_P``` stage runs the first internal step a total of 5 times and the second 3 times, so the expanded stage is ```[restricted_N_P, restricted_N_P_analytic, restricted_N_P, restricted_N_P_analytic, restricted_N_P, restricted_N_P_analytic, restricted_N_P, restricted_N_P]```.

