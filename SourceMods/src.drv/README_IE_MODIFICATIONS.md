# Driver (src.drv) Source Modifications for IE Implementation

This directory contains modifications to CIME's driver code to enable Interactive Ensemble (IE) multi-instance support in CESM2.2.2_IE.

## Files Modified

### 1. cime_comp_mod.F90
**Purpose**: Main CIME driver component module
**Changes Made**:
  - Line 2186: Added ocean instance index calculation: `eoi = mod((exi-1),num_inst_ocn) + 1`
  - Line 2189: Fixed hardcoded `ocn(1)` reference → `ocn(eoi)` in seq_flux_ocnalb_mct call
  - Line 3864: Added ocean instance index calculation: `eoi = mod((exi-1),num_inst_ocn) + 1`
  - Line 3867: Fixed hardcoded `ocn(1)` reference → `ocn(eoi)` in seq_flux_ocnalb_mct call

**Why**: The original code assumed single ocean instance (num_inst_ocn = 1). With multi-instance support, the proper ocean instance index must be calculated using modulo arithmetic to map coupling instances to available ocean instances.

**Impact**: Enables proper multi-instance coupling to single or multiple ocean instances using modulo-based mapping.

### 2. prep_ocn_mod.F90
**Purpose**: Ocean preparation module handling flux merging and ocean input preparation
**Status**: Copy placed in SourceMods, ready for future enhancements
**Future Changes Planned**:
  - Ocean output broadcasting for 1-to-N instance coupling
  - Enhanced documentation of merge strategy
  - Possible namelist exposure of x2o_average flag

**Note**: Current version already supports multi-instance merging with x2o_average flag:
  - If num_inst_ocn = 1 and num_inst_max > 1: averages all instance fluxes to ocean
  - If num_inst_ocn = num_inst_max: uses modulo mapping for 1-to-1 coupling

## Configuration

To enable IE with these modifications:

1. **Set preprocessor defines at build time**:
   ```bash
   export NUM_COMP_INST_ATM=10
   export NUM_COMP_INST_LND=10
   export NUM_COMP_INST_ICE=10
   export NUM_COMP_INST_OCN=1
   ```

2. **Automatic behavior**:
   - x2o_average automatically set to TRUE when num_inst_ocn=1 and num_inst_max>1
   - Flux merging handled automatically in prep_ocn_mod.F90
   - All 10 ATM, LND, ICE instances have their fluxes averaged and sent to single ocean

## Testing Status

- [x] Fixed hardcoded ocn(1) references
- [ ] Ocean output broadcast mechanism (planned)
- [ ] x2o_average configuration (in progress)
- [ ] Multi-instance end-to-end testing

## References

- CESM2 Documentation: http://www.cesm.ucar.edu
- Original Issue: Multiple ocean instances not supported due to hardcoded ocn(1) refs
- CCSM4_IE Implementation: /Users/Natalie/CESM2_IE/ccsm4_0_IE/ (reference)
