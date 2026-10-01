# DUNE Event Preprocessing 2026 
## Step 1: 

This step processes one event from a 2026 HD antineutrino reco2 file and creates a ROOT TTree at recoEnergy/WC. The pixel-map and HDF5 stages have not been run.

Files used
| Purpose | Path |
|---|---|
| 2026 input data | /mnt/ironwolf_14t2/users/jiaxi/dune2026mc/hd_anue/anue_dune10kt_1x2x6_1404_731_20230825T191639Z_gen_g4_detsim_hitreco_20260723T150214Z_reco2.root |
| Original analyzer source shared by Bin (copied while available) | /home/binzhang/disk_binzhang/larsoft_test/Preprocessing_Alejandro/RecoEnergyS/RecoEnergyS_module.cc |
| Compiled source copy | /home/nataliema/recoenergy_one_sl7/srcs/dunereco/dunereco/FDSensOpt/RecoEnergySOne/RecoEnergyS_module.cc |
| CMake file for this module | /home/nataliema/recoenergy_one_sl7/srcs/dunereco/dunereco/FDSensOpt/RecoEnergySOne/CMakeLists.txt |
| Compiled plugin | /home/nataliema/recoenergy_one_sl7/build_slf7.x86_64/dunereco/slf7.x86_64.e26.prof/lib/libRecoEnergyS_module.so |
| One-event FHiCL | /home/nataliema/transformercvn_fcl_overrides/recoenergys_2026_one.fcl |
| Reusable run script | /home/nataliema/run_recoenergy_2026_one.sh |
| Output TTree ROOT file | /home/nataliema/recoenergy_2026_one_event/RecoEnergyS_2026_one.root |
| Run log | /home/nataliema/recoenergy_2026_one_event.log |
| Build log | /home/nataliema/recoenergy_sl7_build.log |


## New FHiCL file

The reco2 input uses `dune10kt_v5_1x2x6` geometry. The original `recoenergys.fcl` selected v2 geometry and ran additional producers. For this one-event test, I used v5 geometry and scheduled only the `RecoEnergyS` analyzer. The input already contains `hitfd` hits, `pandora` clusters and vertices, `pandoraShower` showers, `pandoraTrack` tracks, and the energy reconstruction products.

I saved the following file as `/home/nataliema/transformercvn_fcl_overrides/recoenergys_2026_one.fcl`:

```fcl
#include "services_dune.fcl"
#include "calorimetry_dune10kt.fcl"
#include "tools_dune.fcl"

process_name: RecoEnergySOne2026

services:
{
  TFileService: {
    fileName: "RecoEnergyS_2026_one.root"
    closeFileFast: false
  }
  @table::dunefd_reco_services
  TimeTracker: {}
  RandomNumberGenerator: {}
  MemoryTracker: {}
  message: @local::dune_message_services_prod
  FileCatalogMetadata: @local::art_file_catalog_mc
  @table::dunefd_1x2x6_simulation_services
  Geometry: @local::dune10kt_1x2x6_v2_geo
}

services.Geometry: {
  service_type: "Geometry"
  Name: "dune10kt_v5_1x2x6"
  GDML: "dune10kt_v5_refactored_1x2x6.gdml"
  DisableWiresInG4: true
  SortingParameters: { tool_type: GeoObjectSorterAPA ChannelsPerOpDet: 1 }
  SurfaceY: 147828
}
services.AuxDetGeometry: @local::dune10kt_1x2x6_v5_refactored_auxdet_geo
services.LArG4Detector.gdmlFileName_: "dune10kt_v5_refactored_1x2x6.gdml"
services.BackTrackerService:
{
  BackTracker:
  {
    G4ModuleLabel: "largeant"
    SimChannelModuleLabel: "largeant"
    MinimumHitEnergyFraction: 0.1
  }
}
services.ParticleInventoryService:
{
  ParticleInventory:
  {
    G4ModuleLabel: "largeant"
    EveIdCalculator: "EmEveIdCalculator"
  }
}

source: {
  module_type: RootInput
  maxEvents: 1
}

physics: {
  analyzers: {
    recoEnergy: {
      module_type: "RecoEnergyS"
      CalorimetryAlg: @local::dune10kt_calorimetryalgmc
      HitsModuleLabel: "hitfd"
      ClusterModuleLabel: "pandora"
      ShowerModuleLabel: "pandoraShower"
      TrackModuleLabel1: "pmtrack"
      TrackModuleLabel2: "pandoraTrack"
      RawDigitModuleLabel: "daq"
      WireModuleLabel: "caldata"
      GeantModuleLabel: "largeant"
      VertexModuleLabel: "pandora"
      PFParticleModuleLabel: "pandora"
      PandoraNuVertexModuleLabel: "pandora"
      MCGenModuleLabel: "generator"
      GlobalWireMethod: 2
      EnergyRecoNueLabel: "energyrecnue"
      EnergyRecoNumuLabel: "energyrecnumu"
      EnergyRecoNuelseLabel: "energyrecnc"
      IsVD: false
    }
  }
  myana: [recoEnergy]
  end_paths: [myana]
}
```

The `services` table has a v2 geometry entry from the reference configuration. The later `services.Geometry` assignment sets the v5 geometry used in this run. I set `ClusterModuleLabel` to `pandora` because this input contains `pandora` clusters. The `tools_dune.fcl` include is needed to initialize one of the services.

## Build

I compiled `RecoEnergyS_module.cc` in `/home/nataliema/recoenergy_one_sl7`. The host runs Ubuntu 22.04, and the LArSoft build uses `slf7.x86_64.e26.prof`, so I built inside a Scientific Linux 7 Apptainer image. This was a one-time build; I do not rebuild for each event.

The work area uses LArSoft `v10_23_00` and mrb `v6_09_13`. Its `srcs/CMakeLists.txt` includes `dunereco`, and `dunereco/dunereco/FDSensOpt/CMakeLists.txt` includes `RecoEnergySOne`. With those files already in place, I used the following build commands:

```bash
APP=/cvmfs/oasis.opensciencegrid.org/mis/apptainer/current/bin/apptainer
IMG=/cvmfs/singularity.opensciencegrid.org/fermilab/fnal-dev-sl7:latest

"$APP" exec --cleanenv -B /cvmfs -B "$HOME:$HOME" "$IMG" \
  /bin/bash --noprofile --norc <<'BASH'
export UPS_OVERRIDE='-H Linux64bit+3.10-2.17'
export PRODUCTS=/cvmfs/larsoft.opensciencegrid.org/products:/cvmfs/larsoft.opensciencegrid.org/packages:/cvmfs/dune.opensciencegrid.org/products/dune:/cvmfs/fermilab.opensciencegrid.org/products/common/db
source /cvmfs/larsoft.opensciencegrid.org/products/setup
setup larsoft v10_23_00 -q e26:prof || exit 1
setup mrb v6_09_13 || exit 1

DEV="$HOME/recoenergy_one_sl7"
source "$DEV/localProducts_larsoft_v10_23_00_e26_prof/setup" || exit 1
export PERL5LIB="/cvmfs/larsoft.opensciencegrid.org/products/mrb/v6_03_00/slf7.x86_64/CPAN/lib/perl5:${PERL5LIB:-}"
export CETMODULES_DIR=/cvmfs/larsoft.opensciencegrid.org/products/cetmodules/v3_27_04/slf7.x86_64
cd "$DEV" || exit 1
source "$MRB_DIR/libexec/mrbSetEnv" > "$HOME/recoenergy_sl7_env.log" 2>&1 || exit 1

mrb build -j4 > "$HOME/recoenergy_sl7_build.log" 2>&1
status=$?
echo "build_status=$status"
find "$DEV" -name 'libRecoEnergyS_module.so' -print
exit "$status"
BASH
```

The build returned `build_status=0` and produced `/home/nataliema/recoenergy_one_sl7/build_slf7.x86_64/dunereco/slf7.x86_64.e26.prof/lib/libRecoEnergyS_module.so`.

## Run one event

I saved the following script as `/home/nataliema/run_recoenergy_2026_one.sh`:

```bash
#!/usr/bin/env bash
APP=/cvmfs/oasis.opensciencegrid.org/mis/apptainer/current/bin/apptainer
IMG=/cvmfs/singularity.opensciencegrid.org/fermilab/fnal-dev-sl7:latest

"$APP" exec --cleanenv -B /cvmfs -B "$HOME:$HOME" \
  -B /mnt/ironwolf_14t2:/mnt/ironwolf_14t2 "$IMG" \
  /bin/bash --noprofile --norc <<'INNER'
export UPS_OVERRIDE='-H Linux64bit+3.10-2.17'
export PRODUCTS=/cvmfs/larsoft.opensciencegrid.org/products:/cvmfs/larsoft.opensciencegrid.org/packages:/cvmfs/dune.opensciencegrid.org/products/dune:/cvmfs/fermilab.opensciencegrid.org/products/common/db
source /cvmfs/larsoft.opensciencegrid.org/products/setup
setup larsoft v10_23_00 -q e26:prof || exit 1
setup mrb v6_09_13 || exit 1

DEV="$HOME/recoenergy_one_sl7"
source "$DEV/localProducts_larsoft_v10_23_00_e26_prof/setup" || exit 1
export PERL5LIB="/cvmfs/larsoft.opensciencegrid.org/products/mrb/v6_03_00/slf7.x86_64/CPAN/lib/perl5:${PERL5LIB:-}"
export CETMODULES_DIR=/cvmfs/larsoft.opensciencegrid.org/products/cetmodules/v3_27_04/slf7.x86_64
cd "$DEV" || exit 1
source "$MRB_DIR/libexec/mrbSetEnv" > "$HOME/recoenergy_sl7_env.log" 2>&1 || exit 1

DUNE=/cvmfs/dune.opensciencegrid.org/products/dune
for product in dunecore dunereco dunedataprep dunesim dunesw; do
    dir="$DUNE/$product/v10_23_00d00/fcl"
    [[ -d "$dir" ]] && export FHICL_FILE_PATH="$dir:${FHICL_FILE_PATH:-}"
done
export FW_SEARCH_PATH="$DUNE/dunecore/v10_23_00d00/gdml:$DUNE/dune_pardata/v01_03_02:${FW_SEARCH_PATH:-}"

dune_libpath=""
for dir in "$DUNE"/*/v10_23_00d00/slf7.x86_64.e26.prof/lib; do
    [[ -d "$dir" ]] && dune_libpath="$dir:$dune_libpath"
done
local_lib="$DEV/build_slf7.x86_64/dunereco/slf7.x86_64.e26.prof/lib"
export CET_PLUGIN_PATH="$local_lib:$dune_libpath${CET_PLUGIN_PATH:-}"
export LD_LIBRARY_PATH="$local_lib:$dune_libpath${LD_LIBRARY_PATH:-}"

FCL="$HOME/transformercvn_fcl_overrides/recoenergys_2026_one.fcl"
INPUT=/mnt/ironwolf_14t2/users/jiaxi/dune2026mc/hd_anue/anue_dune10kt_1x2x6_1404_731_20230825T191639Z_gen_g4_detsim_hitreco_20260723T150214Z_reco2.root
OUT="$HOME/recoenergy_2026_one_event"
mkdir -p "$OUT" || exit 1
cd "$OUT" || exit 1

lar -n 1 -c "$FCL" -s "$INPUT" > "$HOME/recoenergy_2026_one_event.log" 2>&1
status=$?
echo "lar_status=$status"
tail -n 35 "$HOME/recoenergy_2026_one_event.log"
ls -lh "$OUT"

if [[ $status -eq 0 ]]; then
    root -l -b -q -e 'TFile f("RecoEnergyS_2026_one.root"); auto* t=f.Get<TTree>("recoEnergy/WC"); std::cout << "WC_ENTRIES=" << (t ? t->GetEntries() : -1) << std::endl;' 2>&1 | tail -n 12
fi
exit "$status"
INNER
```

I ran it with:

```bash
bash "$HOME/run_recoenergy_2026_one.sh"
```

This command uses my paths on tau-neutrino. If you run it from your own account, first copy the FHiCL file and run script to your account, and build the RecoEnergyS plugin in your own LArSoft work area. Then update DEV, FCL, INPUT, and OUT in the script, and check that you can read the input ROOT file. The script will not run unchanged from another account.

The script uses a fixed output filename. Running it again can replace the previous ROOT file.

## Result

My run returned:

```text
lar_status=0
Loop Hits: 3507
Reco nuE: 2.49334
Reco lepE: 0.918352
Reco hadE: 1.57499
Art has completed and will exit with status 0.
WC_ENTRIES=1
```

The output ROOT file is about 2.0 MB. `WC_ENTRIES=1` confirms that `recoEnergy/WC` contains one event. I have not checked the reconstructed energy values against truth values yet.


# Step 2

This step reads the one-event `recoEnergy/WC` tree from Step 1 and makes a pixel-map ROOT file. Alejandro's macro is used. The directory had moved from the path used in Step 1 to `/mnt/ironwolf_14t2/users/binzhang/preprocessing/Preprocessing_Alejandro`.

## Files used

| Purpose | Path |
|---|---|
| Input WC ROOT file | `/home/nataliema/recoenergy_2026_one_event/RecoEnergyS_2026_one.root` |
| Original pixel-map macro | `/mnt/ironwolf_14t2/users/binzhang/preprocessing/Preprocessing_Alejandro/make_text_file_to_root_trks_shws.C` |
| Local diagnostic macro | `/home/nataliema/recoenergy_2026_one_event/step2_hitcenter_diagnostic/make_text_file_to_root_trks_shws.C` |
| Pixel-map ROOT output | `/home/nataliema/recoenergy_2026_one_event/step2_hitcenter_diagnostic/pixelmap_2026_one_nue_hitcenter_diagnostic.root` |
| Run log | `/home/nataliema/recoenergy_2026_one_event/step2_hitcenter_diagnostic/pixelmap_2026_one_nue_hitcenter_diagnostic.log` |

## Changes for this one-event test

The original macro reads the input tree and sees one event, but its `nue` selection sets `fiducial_cut=1` and skips event 73101. This flag is based on hit wire and tick conditions in the macro. It does not by itself establish that the true vertex is outside the detector fiducial volume.

For this event, the macro also receives invalid PF-vertex map coordinates near `-9999`. The input hits have nonzero charge (`all_abs_charge=908671` in the diagnostic output), but zero hits land within the 400×280 pixel-map window when it is centered on those invalid coordinates. The empty tree initially contained one zero-charge placeholder for each map.

I made a local copy of the macro and changed two things for this diagnostic test: I did not skip the event on `fiducial_cut`, and I used the arithmetic mean of the hit global wire and tick in each plane as the map center. The macro already accumulates `sum_wire`, `sum_time`, and `nhits_count`. The original file was not edited. This output is a pipeline test for one chosen event, not a sample selected by the original `nue` cuts or centered on a reconstructed PF vertex.

The following reproduces these two changes in a local copy:

```bash
ORIG=/mnt/ironwolf_14t2/users/binzhang/preprocessing/Preprocessing_Alejandro/make_text_file_to_root_trks_shws.C
WORK="$HOME/recoenergy_2026_one_event/step2_hitcenter_diagnostic"
mkdir -p "$WORK"

python3 - "$ORIG" "$WORK/make_text_file_to_root_trks_shws.C" <<'PY'
from pathlib import Path
import re
import sys

src, dst = map(Path, sys.argv[1:])
text = src.read_text()

cut = 'if (type == "nue" && fiducial_cut) continue;'
assert text.count(cut) == 1
text = text.replace(
    cut,
    'std::cout << "DIAGNOSTIC nhits=" << nhits '
    '<< " fiducial_cut=" << fiducial_cut << std::endl;\n'
    '        // One-event diagnostic: do not skip on the nue fiducial cut.',
    1,
)

wire_pattern = r'(?m)^([ \t]*)mean_wire\[ii\]\s*=\s*pfVtxGlobalWire\[[^\n;]+\];'
time_pattern = r'(?m)^([ \t]*)mean_time\[ii\]\s*=\s*pfVtxGlobalTick\[[^\n;]+\];'

def replace_wire(match):
    indent = match.group(1)
    return (
        f'{indent}if (nhits_count[ii] == 0) std::abort();\n'
        f'{indent}mean_wire[ii] = int(round(sum_wire[ii] / nhits_count[ii]));'
    )

def replace_time(match):
    indent = match.group(1)
    return (
        f'{indent}mean_time[ii] = int(round(sum_time[ii] / nhits_count[ii]));\n'
        f'{indent}std::cout << "HIT_CENTER plane=" << ii '
        f'<< " wire=" << mean_wire[ii] '
        f'<< " tick=" << mean_time[ii] << std::endl;'
    )

text, nw = re.subn(wire_pattern, replace_wire, text)
text, nt = re.subn(time_pattern, replace_time, text)
assert (nw, nt) == (1, 1), (nw, nt)
dst.write_text(text)
print(f"Created {dst}")
PY
```

Run the local macro on the same one-event input and check that the output contains nonzero charge:

```bash
INPUT="$HOME/recoenergy_2026_one_event/RecoEnergyS_2026_one.root"
OUTPUT="$WORK/pixelmap_2026_one_nue_hitcenter_diagnostic.root"
LOG="$WORK/pixelmap_2026_one_nue_hitcenter_diagnostic.log"

root -l -b -q "$WORK/make_text_file_to_root_trks_shws.C(\"$INPUT\",\"$OUTPUT\",\"nue\")" > "$LOG" 2>&1
echo "macro_exit=$?"
grep -E 'Entries:|Ievent:|DIAGNOSTIC|HIT_CENTER|Error|Running time' "$LOG" | tail -20
ls -lh "$OUTPUT"

export PIXELMAP_OUTPUT="$OUTPUT"
root -l -b -q -e 'TFile f(gSystem->Getenv("PIXELMAP_OUTPUT")); auto* t=f.Get<TTree>("pixelmap"); std::cout << "ROWS=" << (t ? t->GetEntries() : -1) << " NONZERO_CHARGE=" << (t ? t->GetEntries("wire_charge!=0") : -1) << " NONZERO_CORRCHARGE=" << (t ? t->GetEntries("wire_corrcharge!=0") : -1) << std::endl;' 2>&1 | tail -n 12
```

## Result

My one-event run returned:

```text
macro_exit=0
Entries: 1
-->0 , Ievent: 73101
HIT_CENTER plane=0 wire=2123 tick=4381
HIT_CENTER plane=1 wire=1617 tick=4380
HIT_CENTER plane=2 wire=1749 tick=4406
ROWS=4047 NONZERO_CHARGE=3996 NONZERO_CORRCHARGE=3996
```

The output ROOT file is about 48 KB. The 4047 pixel-map rows come from the same one input event; they are not 4047 events. The nonzero charge counts show that the diagnostic map contains hit information. To use the original `nue` selection and PF-vertex centering, a different 2026 event with valid PF-vertex coordinates and a passing fiducial cut is needed.


# Step 3

This step converts the diagnostic pixel-map ROOT file to an intermediate HDF5 file, then uses `preprocess2.py` from the shared preprocessing directory to produce the sparse HDF5 arrays. It processes the same single event. The 400×280 image shape matches the `nue` pixel-map macro.

## Files used

| Purpose | Path |
|---|---|
| Pixel-map ROOT input | `/home/nataliema/recoenergy_2026_one_event/step2_hitcenter_diagnostic/pixelmap_2026_one_nue_hitcenter_diagnostic.root` |
| Alejandro's `root2hdf5` executable | `/home/ayankelevich/.local/bin/root2hdf5` |
| Intermediate HDF5 output | `/home/nataliema/recoenergy_2026_one_event/step3_diagnostic/pixelmap_2026_one_nue_hitcenter_diagnostic.root.h5` |
| Reference HDF5 preprocessing script | `/mnt/ironwolf_14t2/users/binzhang/preprocessing/Preprocessing_Alejandro/preprocess2.py` |
| Local one-event script copy | `/home/nataliema/recoenergy_2026_one_event/step3_diagnostic/preprocess2_one.py` |
| Final diagnostic HDF5 output | `/home/nataliema/recoenergy_2026_one_event/step3_diagnostic/pixelmap_2026_one_nue_hitcenter_diagnostic_final.h5` |

## Convert the ROOT file to intermediate HDF5

The `root2hdf5` executable imports `src.root2hdf5` from Alejandro's Python 3.10 site-packages. I supplied that directory in `PYTHONPATH` and converted only the one-event ROOT file. I did not run the reference `convert_h5.sh`, which loops over many atmospheric-neutrino files.

```bash
ROOT_FILE="$HOME/recoenergy_2026_one_event/step2_hitcenter_diagnostic/pixelmap_2026_one_nue_hitcenter_diagnostic.root"
OUT_DIR="$HOME/recoenergy_2026_one_event/step3_diagnostic"
INTERMEDIATE_H5="$OUT_DIR/pixelmap_2026_one_nue_hitcenter_diagnostic.root.h5"
CONVERTER=/home/ayankelevich/.local/bin/root2hdf5

mkdir -p "$OUT_DIR"
if [[ ! -e "$INTERMEDIATE_H5" ]]; then
  PYTHONPATH=/home/ayankelevich/.local/lib/python3.10/site-packages \
    "$CONVERTER" -i "$ROOT_FILE" -o "$INTERMEDIATE_H5" -t pixelmap \
    > "$OUT_DIR/root2hdf5_retry.log" 2>&1
  echo "convert_status=$?"
  tail -n 25 "$OUT_DIR/root2hdf5_retry.log"
fi
```

The conversion returned `retry_status=0` in the original run and produced a 1.6 MB file. Its HDF5 root contains a compound dataset named `pixelmap` with 4047 rows. The dataset name matters for the next step.

## Produce the final HDF5 arrays

The reference `preprocess2.py` expects a compound dataset named `ArrayOfStructures`, while this `root2hdf5` output names it `pixelmap`. Its command-line input loop also starts at `sys.argv[1]`, which includes the output path. I copied the script locally and changed only these two references. The shared file remains the reference source.

```bash
BASE=/mnt/ironwolf_14t2/users/binzhang/preprocessing/Preprocessing_Alejandro
OUT_DIR="$HOME/recoenergy_2026_one_event/step3_diagnostic"
INPUT_H5="$OUT_DIR/pixelmap_2026_one_nue_hitcenter_diagnostic.root.h5"
FINAL_H5="$OUT_DIR/pixelmap_2026_one_nue_hitcenter_diagnostic_final.h5"
LOCAL_SCRIPT="$OUT_DIR/preprocess2_one.py"

python3 - "$BASE/preprocess2.py" "$LOCAL_SCRIPT" <<'PY'
from pathlib import Path
import sys

source, target = map(Path, sys.argv[1:3])
script = source.read_text()
old_table = "['ArrayOfStructures']"
old_args = "for arg in sys.argv[1:]:"
assert script.count(old_table) == 1
assert script.count(old_args) == 1
script = script.replace(old_table, "['pixelmap']", 1)
script = script.replace(old_args, "for arg in sys.argv[2:]:", 1)
target.write_text(script)
print("Created:", target)
PY

if [[ ! -e "$FINAL_H5" ]]; then
  PYTHONPATH=/home/ayankelevich/.local/lib/python3.10/site-packages \
    /usr/bin/python3 "$LOCAL_SCRIPT" "$FINAL_H5" "$INPUT_H5" \
    > "$OUT_DIR/preprocess2_one.log" 2>&1
  echo "preprocess_status=$?"
  tail -n 35 "$OUT_DIR/preprocess2_one.log"
fi
```

I used `/usr/bin/python3` here because it has `tables 3.7.0`; my default Miniconda Python did not have `tables`. The run returned `preprocess_status=0` and printed `1` processed file and `1` event. The `NaturalNameWarning` messages refer to HDF5 node names containing periods, such as `mc.png_label`.

Check the final file:

```bash
FINAL_H5="$HOME/recoenergy_2026_one_event/step3_diagnostic/pixelmap_2026_one_nue_hitcenter_diagnostic_final.h5"

/usr/bin/python3 - "$FINAL_H5" <<'PY'
import sys
import numpy as np
import tables

with tables.open_file(sys.argv[1], "r") as file:
    root = file.root
    shape = root.cvnmap_shape.read()
    charges = root.cvnmap_value.read()
    print("CVNMAP_SHAPE=", shape.tolist())
    print("EVENT_ROWS=", root.trueE.nrows)
    print("SPARSE_PIXELS=", len(charges))
    print("NONZERO_CHARGE=", np.count_nonzero(charges))
    print("PRONG_MASK_SHAPE=", root.input_png3d_pad_mask.shape)
    assert shape.tolist() == [1, 3, 400, 280]
    assert root.trueE.nrows == 1
    assert np.count_nonzero(charges) > 0
PY
```

## Result and limits

The validated output is:

```text
CVNMAP_SHAPE= [1, 3, 400, 280]
EVENT_ROWS= 1
SPARSE_PIXELS= 2061
NONZERO_CHARGE= 2058
PRONG_MASK_SHAPE= (1, 20)
```

`SPARSE_PIXELS` counts event-map pixels, not events. The earlier ROOT tree's 4047 rows also include prong maps. The final preprocessing script printed `Unknown pdg: -999` for multiple prongs, so those prong truth labels are missing for this event. This output demonstrates that the one-event pipeline creates nonempty event images; it should not be used as a standard selected `nue` sample or as a training example with reliable prong labels. The diagnostic macro bypasses the original `nue` fiducial cut and centers maps on hit means because the PF-vertex map coordinates were invalid for this event.


