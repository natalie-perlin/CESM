# Configure CESM 2.2.2 on Derecho

This README summarizes the steps used to configure, build, and submit a CESM 2.2.2 control run on the NCAR Derecho system (`derecho.hpc.ucar.edu`).

**Original notes:** January 20, 2026  
**Updated:** May 25, 2026; June 2, 2026

> **Note**
> Commands shown across multiple lines in this README may be copied as multi-line shell commands. If converting from older notes where a command was visually wrapped, enter it as one logical command.

## Assumptions

This workflow assumes:

- You are working on NCAR Derecho.
- You have access to a valid NCAR project/account, shown below as `UMIA0045`.
- You are using CESM 2.2.2 with CIME updated to `cime5.8.37` for Derecho compatibility.
- You have a site-local CIME machine configuration for Derecho, provided through `--extra-machines-dir`.
- You will edit placeholder paths such as `<YOUR/CASE/DIR>`, `<CESM_EXPT_DIR>`, and `<CASE_DIR>` for your experiment.

## 1. Check out CESM 2.2.2

Clone the CESM 2.2.2 release branch:

```bash
git clone -b release-cesm2.2.2 https://github.com/ESCOMP/CESM.git cesm2.2.2
cd cesm2.2.2

export CESM_ROOT=$PWD
export CIMEROOT=$CESM_ROOT/cime
```

## 2. Update the CIME external version

For Derecho compatibility, use CIME branch `cime5.8.37` instead of the default `cime5.8.32.9` referenced by this CESM release.

Edit `Externals.cfg` directly or use `sed`:

```bash
sed -i 's/cime5\.8\.32\.9/cime5.8.37/g' Externals.cfg
```

Alternative method:

1. Check out all externals first, as shown in the next section.
2. Then switch the CIME checkout manually:

```bash
cd $CESM_ROOT/cime
git checkout cime5.8.37
```

## 3. Check out CESM externals

From `$CESM_ROOT`:

```bash
./manage_externals/checkout_externals
```

## 4. Add Derecho CIME machine configuration

Create a local CIME machine-configuration directory under the CESM checkout:

```bash
mkdir -p $CESM_ROOT/cime_config
```

Copy a Derecho-compatible CIME machine-configuration template into place. For example:

```bash
cp -vr /path/to/machines_template $CESM_ROOT/cime_config/machines
```

The resulting directory should contain at least:

```text
$CESM_ROOT/cime_config/machines/config_machines.xml
$CESM_ROOT/cime_config/machines/config_batch.xml
```

Edit:

```text
$CESM_ROOT/cime_config/machines/config_machines.xml
```

Adjust machine-specific paths as needed, especially:

- `CIME_OUTPUT_ROOT`
- `DOUT_S_ROOT`

These should reflect your experiment layout, available disk space, and desired short-term archive location.

## 5. Create a new CESM case

From `$CESM_ROOT`, create a new case. Edit the case path, resolution, compset, project/account, and other arguments as needed.

Example:

```bash
cd $CESM_ROOT

$CIMEROOT/scripts/create_newcase \
  --case <YOUR/CASE/DIR> \
  --res f09_g17 \
  --compset B1850 \
  --project UMIA0045 \
  --machine derecho \
  --extra-machines-dir $CESM_ROOT/cime_config/machines \
  2>&1 | tee log.create.newcase.001
```

This creates the case directory:

```text
<YOUR/CASE/DIR>
```

For reproducibility, it is useful to save the `create_newcase` command in a script, for example:

```bash
cat > $CIMEROOT/scripts/newcase.sh <<'EOF_SCRIPT'
#!/bin/bash

$CIMEROOT/scripts/create_newcase \
  --case <YOUR/CASE/DIR> \
  --res f09_g17 \
  --compset B1850 \
  --project UMIA0045 \
  --machine derecho \
  --extra-machines-dir $CESM_ROOT/cime_config/machines
EOF_SCRIPT

chmod +x $CIMEROOT/scripts/newcase.sh
```

## 6. Prepare the case setup

Move to the new case directory:

```bash
cd <YOUR/CASE/DIR>
export CASEROOT=$PWD
```

### 6.1 Check and optionally change run/archive paths

Query the current case root, run directory, and short-term archive root:

```bash
./xmlquery CASEROOT RUNDIR DOUT_S_ROOT
```

A common layout is to keep `CASEROOT` under `/glade/work/${USER}` while placing `RUNDIR` under `${SCRATCH}` for better I/O performance.

Create a run directory, then update `RUNDIR`:

```bash
mkdir -p ${SCRATCH}/<CESM_EXPT_DIR>/<CASE_DIR>/run

cd $CASEROOT
./xmlchange RUNDIR=${SCRATCH}/<CESM_EXPT_DIR>/<CASE_DIR>/run
```

Verify the change:

```bash
./xmlquery RUNDIR
```

### 6.2 Reduce WAV task count

The default wave-model task count may be larger than needed. In this setup, `NTASKS_WAV=36` was found to work well:

```bash
./xmlchange NTASKS_WAV=36
```

### 6.3 Limit MARBL ocean diagnostics

To prevent MARBL from writing hundreds of ocean biogeochemistry variables, create an empty `ecosys_diagnostics` file:

```bash
echo ' ' > SourceMods/src.pop/ecosys_diagnostics
```

### 6.4 Optional: add daily POP momentum-flux output

If daily wind-stress output is needed, add the POP fields:

```text
TAUX_2
TAUY_2
```

Place the relevant POP source modifications under:

```text
${CASEROOT}/SourceMods/src.pop/
```

The source modifications should include:

```text
forcing.F90
gx1v7_tavg_contents
```

Example copy command:

```bash
cd $CASEROOT
cp -pv /path/to/adapted/SourceMods/src.pop/* ${CASEROOT}/SourceMods/src.pop/.
```

In the original workflow, these changes were adapted from Sara Larson's daily POP tau workflow and from an existing CESM2.2.2 experiment tree.

### 6.5 Run `case.setup`

```bash
cd $CASEROOT
./case.setup 2>&1 | tee log.case.setup.001
```

## 7. Build the case

### 7.1 Clean any previous build

If an old or partial build exists, clean it first:

```bash
./case.build --clean
```

### 7.2 Reset and inspect Derecho modules

Use the default Derecho module environment for the build:

```bash
module reset
module list
```

The working module environment in the original setup was:

```text
1) ncarenv/24.12
2) craype/2.7.31
3) intel/2024.2.1
4) ncarcompilers/1.0.0
5) libfabric/1.15.2.0
6) cray-mpich/8.1.29
7) hdf5/1.12.3
8) netcdf/4.9.2
```

### 7.3 Set NetCDF paths

Verify that `nc-config` and `nf-config` are available from the loaded modules:

```bash
nc-config --prefix
nf-config --prefix
```

Example paths from the original setup:

```text
/glade/u/apps/derecho/24.12/spack/opt/spack/netcdf/4.9.2/packages/netcdf-c/4.9.2/oneapi/2024.2.1/6ont
/glade/u/apps/derecho/24.12/spack/opt/spack/netcdf/4.9.2/packages/netcdf-fortran/4.6.1/oneapi/2024.2.1/fq5j
```

Set the NetCDF environment variables:

```bash
export NETCDF_C_PATH="$(nc-config --prefix)"
export NETCDF_FORTRAN_PATH="$(nf-config --prefix)"
export NETCDF_PATH="$(nc-config --prefix)"
```

### 7.4 Update the POP namelist

Add the following line to the bottom of `user_nl_pop`:

```text
n_tavg_streams = 3
```

### 7.5 Build with `qcmd`

```bash
qcmd -A UMIA0045 -- ./case.build 2>&1 | tee log.case.build.002
```

## 8. Configure runtime parameters and submit the case

### 8.1 Set run length, resubmission, archive, and wallclock settings

Query and adjust runtime settings:

```bash
./xmlquery EXEROOT

./xmlquery STOP_OPTION,STOP_N
./xmlchange STOP_OPTION=nmonths,STOP_N=6

./xmlquery RESUBMIT
./xmlchange RESUBMIT=1

./xmlquery DOUT_S_ROOT
./xmlchange DOUT_S=TRUE

./xmlquery JOB_WALLCLOCK_TIME
./xmlchange --subgroup case.run JOB_WALLCLOCK_TIME=02:00:00
./xmlchange --subgroup case.st_archive JOB_WALLCLOCK_TIME=00:20:00
```

The example above configures:

- A 6-month run segment.
- One automatic resubmission.
- Short-term archiving enabled.
- A 2-hour wallclock limit for `case.run`.
- A 20-minute wallclock limit for `case.st_archive`.

### 8.2 Submit the case

```bash
./case.submit >& log.case.submit.001 &
```

## Useful verification commands

Use these commands to inspect the active case configuration:

```bash
./xmlquery CASEROOT RUNDIR DOUT_S_ROOT
./xmlquery EXEROOT
./xmlquery STOP_OPTION,STOP_N
./xmlquery RESUBMIT
./xmlquery DOUT_S
./xmlquery JOB_WALLCLOCK_TIME
```

Check recent logs:

```bash
ls -ltr log.*
tail -100 log.case.build.002
tail -100 log.case.submit.001
```

## Notes for adapting this README

Before committing this file to a public or shared GitHub repository, review and replace site- or user-specific values, including:

- NCAR project/account names, such as `UMIA0045`.
- Absolute paths under `/glade/work/<user>` or `/glade/campaign/...`.
- Experiment-specific source modifications under `SourceMods`.
- Case names, experiment names, and scratch/archive locations.

Keep the local Derecho machine files in a documented repository location, or provide instructions for users to obtain them separately if they are site-specific.
