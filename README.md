# DUNE-Event-Preprocessing-with-RecoEnergyS-2026-
Step 1: This step processes one event from a 2026 HD antineutrino reco2 file and creates a ROOT TTree at recoEnergy/WC. The pixel-map and HDF5 stages have not been run.

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
