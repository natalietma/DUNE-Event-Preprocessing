# DUNE-Event-Preprocessing-with-RecoEnergyS-2026-
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

The reco2 input uses `dune10kt_v5_1x2x6` geometry. Bin's original `recoenergys.fcl` selected v2 geometry and ran additional producers. For this one-event test, I used v5 geometry and scheduled only the `RecoEnergyS` analyzer. The input already contains `hitfd` hits, `pandora` clusters and vertices, `pandoraShower` showers, `pandoraTrack` tracks, and the energy reconstruction products.

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

I compiled Bin's `RecoEnergyS_module.cc` in `/home/nataliema/recoenergy_one_sl7`. The host runs Ubuntu 22.04, and the LArSoft build uses `slf7.x86_64.e26.prof`, so I built inside a Scientific Linux 7 Apptainer image. This was a one-time build; I do not rebuild for each event.

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

The script uses a fixed output filename. Running it again can replace the previous ROOT file.

## Result

The run on September 30, 2026 returned:

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
