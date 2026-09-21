# BL material maps (T35-T38)

One binary material map per supported tracker geometry:

    BLMaterialMap_T35_BP2030v3_v1.bin   # D121
    BLMaterialMap_T36_BP2030v3_v1.bin   # D111
    BLMaterialMap_T37_BP2030v3_v1.bin   # D112 (and D126)
    BLMaterialMap_T38_BP2030v3_v1.bin   # D113

Each file holds the phi-averaged Geant4 radiation-length density rho(r,z) and the dE/dx quantities the
BrokenLine fits read, on the compile-time 250 x 560 lattice (0.5 cm radial, 1 cm z, |z| <= 280 cm). The
byte format (84-byte little-endian header, provenance text, serialized `blMaterialMap::Map`, sensor
reference positions) is documented in `RecoTracker/PixelTrackFitting/interface/BLMaterialMapFile.h`
and enforced by its reader; every file carries its own provenance (release, geometry, era, beam pipe,
materials, ray sample, fingerprint, sensor census).

Every value is `float("%.6g" % v)` rounded to float32, the floats the retired compiled-in table carried.
T35 is a byte conversion of that table (made in CMSSW_20_1_0_pre2, no value change); T36, T37 and T38
are fresh generations in CMSSW_20_1_X_2026-09-17-2300 (el9_amd64_gcc14) with the recipe of
`RecoTracker/PixelTrackFitting/test/blMaterialMap` (40 jobs x 150000 rays each).

## How a job picks a map

The index key is the fingerprint of the tracker geometry: an FNV-1a hash over the sorted rawIds of all
tracker sensors of the ideal `GeometricDet` (`BLMaterialMapFingerprint.h`), the same for every sourcing
mode (DDD, DD4hep, DB) of one geometry. Tracker versions with the same sensor inventory share the key
(T36, T37, T38 differ only in module placement), so each file also carries its sensors' reference
positions in 0.1 mm buckets, and the producer keeps the candidate whose positions match the job's
within one bucket: sourcing modes move positions by a few nm and can cross a bucket boundary by one
bucket at most, genuine versions differ by at least 6 buckets. The producer then checks the file's
embedded fingerprint and lattice constants. Every failure (no candidate, no position match, two
matches, malformed file) is a `cms::Exception("BLMaterialMap")`: a job never runs silently on the map
of another geometry.

    0x31bac24dfe45a850 T36 2030/v3 1 BLMaterialMap_T36_BP2030v3_v1.bin
    (fingerprint        tag beam-pipe map-version file)

## Adding a tracker geometry Tnn

1. In the release: `RecoTracker/PixelTrackFitting/test/blMaterialMap/blMaterialMapRun.sh <outdir> 40
   150000 Tnn <geometry-cff> <era>` (`--fingerprint-only` first gives the key without rays). The run
   writes `BLMaterialMap_Tnn_BP<bp>_v<n>.bin` (`<n>` = `$MAPVERSION`, default 1) with the reference
   positions appended, and its index line in `<outdir>/BLMaterialMap.index`.
2. Copy the file here and merge the line into `BLMaterialMap.index`.

## Beam-pipe or materials changes

The fingerprint sees tracker sensors only: a beam-pipe or materials change under an unchanged tracker
needs a new map with a bumped version (`MAPVERSION=2`, new beam-pipe tag in the name and index line)
that replaces the tracker's index line. Two files of the same tracker in the index would both match a
job and the producer throws on the ambiguity.
