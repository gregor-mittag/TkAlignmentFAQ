# Conveners archive

`https://groups.cern.ch/group/cms-tracker-alignment-conveners/Lists/Archive/`


# EOS transition of MPproduction area

* migration not finalized due to missing support for HTCondor submission
  * for details, see CERN-IT ticket: https://cern.service-now.com/service-portal/view-request.do?n=RQF0845905
* missing pieces on our side:
  * resync AFS and EOS area -> **make script available**
  * update MPS templates to directly point to new EOS area
  * move AFS area to a backup location, e.g.
    `/afs/cern.ch/cms/CAF/CMSALCA/ALCA_TRACKERALIGN/MP/MPproduction.backup`
    * only temporary, until all dependencies on old AFS location are fixed,
      e.g. all-in-one tool feature `mp = 1234`
  * after moving the AFS area, make the old AFS path a symlink to the new EOS
    area:
	```
	/afs/cern.ch/cms/CAF/CMSALCA/ALCA_TRACKERALIGN/MP/MPproduction/ -> /eos/cms/store/group/alca_millepede/MPproduction
	```


# Various methods to build a TrackerAlignmentRcd

Obviously by running an alignment. But especially at start-up it is desired to
construct a record from various inputs and assumptions.

## Ideal alignment

This can be done with a
[plugin](https://github.com/cms-sw/cmssw/blob/master/Alignment/TrackerAlignment/plugins/CreateTrackerAlignmentRcds.cc)
and the corresponding
[configuration file](https://github.com/cms-sw/cmssw/blob/master/Alignment/TrackerAlignment/test/createTrackerAlignmentRcds_cfg.py).

In the
[configuration file](https://github.com/cms-sw/cmssw/blob/master/Alignment/TrackerAlignment/test/createTrackerAlignmentRcds_cfg.py#L14)
you have to set the following option:
```
process.createIdealTkAlRecords.alignToGlobalTag = False
```
Otherwise, you override (at least parts of) your ideal alignment with the
content of the loaded alignment record (either from the global tag or from an
`ESPrefer`).

Now you have created an sqlite file with an ideal `TrackerAlignmentRcd`,
`TrackerAlignmentErrorExtendedRcd` and `TrackerSurfaceDeformationRcd`.


## Ideal alignment only in parts of the detector

If you want to align the content of your created payloads to the content your
loaded alignment payloads based on DetId matching, you can set the above option
in the
[configuration file](https://github.com/cms-sw/cmssw/blob/master/Alignment/TrackerAlignment/test/createTrackerAlignmentRcds_cfg.py#L14)
to `True`:
```
process.createIdealTkAlRecords.alignToGlobalTag = True
```
This does not make a lot of sense, if the loaded alignment and the new payload
belong to the same geometry, but was quite useful when initial alignment
payloads with the latest strip alignment from the phase-0 detector, but with an
ideal phase-1 pixel. As the DetIds are mutually exclusive for the phase-0 and
phase-1 pixel modules, the above option did exactly that:

Ideal pixel + realistic strip.

If you want to achieve the same for cases where both input and output belong to
the same geometry, you can use this option in the
[configuration file](https://github.com/cms-sw/cmssw/blob/master/Alignment/TrackerAlignment/test/createTrackerAlignmentRcds_cfg.py#L15)
```
process.createIdealTkAlRecords.skipSubDetectors = cms.untracked.vstring("P1PXB", "P1PXEC")
```


## Apply random smearing

Define the input alignment in the
[configuration file](https://github.com/cms-sw/cmssw/blob/master/Alignment/TrackerAlignment/test/Misalignments/createRandomlyMisalignedGeometry_cfg.py)
and the smearing in the
[scenario file](https://github.com/cms-sw/cmssw/blob/master/Alignment/TrackerAlignment/python/Scenarios_cff.py#L3415-L3447)
and run the tool, e.g., like this:
```
cmsRun createRandomlyMisalignedGeometry_cfg.py myScenario=MisalignmentScenario_PhaseI_PseudoAsymptotic
```


# Interplay of GlobalPositionRcd (GPR) and TrackerAlignmentRcd

The `TrackerAlignmentRcd` contains an `Alignments` objects which contains a vector
of `AlignTransform` for each module.

The way the alignment is applied is the following:

* take the position and rotation from the `GlobalPositionRcd` which contains an
  `AlignTransform` object for each subdetector, i.e. among others also for the
  tracker
* take for each module the `AlignTransform` object from the
  `TrackerAlignmentRcd`
  * combine those two transformations [1](https://github.com/cms-sw/cmssw/blob/master/Geometry/TrackingGeometryAligner/interface/GeometryAligner.h#L102-L103)
* these transformations are identical to absolute positions and orientations if
  you apply them to (0,0,0) (position) and (0,0,0) (orientation)
- the position and orientation of the module are then **set** (not moved)
  to the above absolute position and orientation [2](https://github.com/cms-sw/cmssw/blob/master/Geometry/TrackingGeometryAligner/interface/GeometryAligner.h#L111)

To make a long story short:  **real position = alignment**


# How to run a private rereco

The tool used to run the 2017 private rereco is available [here](https://github.com/cms-trackeralign/CosmicsReRecoTools/tree/cruzet-2017-BPIX-Rereco).

Basically one job per RAW input file is submitted and that's it.

If you want to change the input data you just change the file list in [1](https://github.com/cms-trackeralign/CosmicsReRecoTools/blob/cruzet-2017-BPIX-Rereco/all\_cosmics\_cff.py).


# How to manually recenter a geometry

Define the input alignment within the
[configuration file](https://github.com/cms-sw/cmssw/blob/master/Alignment/OfflineValidation/test/GeometryCentering_cfg.py)
and specify the to-be-centered reference system with this parameter:
```
process.TrackerGeometryCompare.setCommonTrackerSystem = "P1PXBBarrel" # for MC
# process.TrackerGeometryCompare.setCommonTrackerSystem = "TOBBarrel"  # for Data
```
Afterwards you run the tool like this:
```
cmsRun GeometryCentering_cfg.py
```


# How to measure ALCARECO rates and event sizes

You can use the script written by Joze Zobec:

https://github.com/cms-trackeralign/ALCARECO-statistics


# Gero's legacy thoughts

https://indico.cern.ch/event/243934/#22-closing-words
https://twiki.cern.ch/twiki/bin/view/CMS/TkAlignmentLegacyThoughts

# How to exclude BPIX hits

In `universalConfigTemplate.py` one has to replace the following snippet (at the
bottom of the file):

```
if setupAlgoMode == "mille":
    import Alignment.MillePedeAlignmentAlgorithm.alignmentsetup.MilleSetup as mille
    mille.setup(process,
                input_files        = readFiles,
                collection         = setupCollection,
                json_file          = setupJson,
                cosmics_zero_tesla = setupCosmicsZeroTesla,
                cosmics_deco_mode  = setupCosmicsDecoMode)
```

with this snippet:

```
if setupAlgoMode == "mille":
    import Alignment.MillePedeAlignmentAlgorithm.alignmentsetup.MilleSetup as mille
    mille.setup(process,
                input_files        = readFiles,
                collection         = setupCollection,
                json_file          = setupJson,
                cosmics_zero_tesla = setupCosmicsZeroTesla,
                cosmics_deco_mode  = setupCosmicsDecoMode)
    process.TrackerTrackHitFilter.commands.remove("keep PXB") # remove BPIX hits
    process.TrackerTrackHitFilter.commands.append("drop PXB") # for real!
```

Note, that it is important to have the two new lines within the if branch,
i.e. mind the indentation.  I am also not sure, if only the second line (marked
with `# for real!`) is sufficient, but using both lines works for sure.


# Switching to generic CPE

If you want to switch from the default template CPE to the generic CPE you have
to change a few configuration parameters.

For the validation you have to set the following two parameters for all validations that use data:

```
ttrhbuilder = WithTrackAngle
usepixelqualityflag = false
```

The default values are listed on the
[All-in-One Tool Twiki](https://twiki.cern.ch/twiki/bin/viewauth/CMS/TkAlAllInOneValidation#Configuration).

For MillePede one has to add another parameter to the `mille.setup()` call:

```
if setupAlgoMode == "mille":
    import Alignment.MillePedeAlignmentAlgorithm.alignmentsetup.MilleSetup as mille
    mille.setup(process,
                input_files        = readFiles,
                collection         = setupCollection,
                json_file          = setupJson,
                cosmics_zero_tesla = setupCosmicsZeroTesla,
                cosmics_deco_mode  = setupCosmicsDecoMode,
                TTRHBuilder        = "WithTrackAngle")
```

There is no need to specify something for the pixel quality flag because this is
only defined when using templates. The method detects if templates are not used
and automatically sets this parameter to `False`, which is not true for the
validation. For the validation one has to explicitly write (as done above):

```
usepixelqualityflag = false
```

If one uses the integrated pixel Lorentz angle calibration one has to change
also the following parameter when using generic CPE:

```
siPixelLA.lorentzAngleLabel = ""
```

This assumes you have included the configuration fragment like this:

```
from Alignment.CommonAlignmentAlgorithm.SiPixelLorentzAngleCalibration_cff \
    import SiPixelLorentzAngleCalibration as siPixelLA
```

By default this label is set to `"fromAlignment"`, i.e. if one wants to override
the Lorentz angle in iterations or validations, one needs to make sure that the
label is consistent.

When one overrides conditions, one often omits the label because it is most of
the time an empty string. If it is not empty, one has to explicitly use it.


# About LA/BP corrections

## How LA are handled together with pixel templates

If templates are used one has to set `DoLorentz` to `True`. This is the default:

https://github.com/cms-sw/cmssw/blob/master/RecoLocalTracker/SiPixelRecHits/python/PixelCPETemplateReco_cfi.py#L16

If the LA is zero (currently true for LA from alignment), we would get a unit
vector in negative z direction here:
https://github.com/cms-sw/cmssw/blob/master/RecoLocalTracker/SiPixelRecHits/src/PixelCPEBase.cc#L524-L532

This is used here:
https://github.com/cms-sw/cmssw/blob/master/RecoLocalTracker/SiPixelRecHits/src/PixelCPEBase.cc#L213-L214

And the members for each DetParam object that are set here:
https://github.com/cms-sw/cmssw/blob/master/RecoLocalTracker/SiPixelRecHits/src/PixelCPEBase.cc#L544-L545

are then finally used here:
https://github.com/cms-sw/cmssw/blob/master/RecoLocalTracker/SiPixelRecHits/src/PixelCPETemplateReco.cc#L399-L400


## Assignment of IOVs

### Question to Nazar Bartosik

You assigned different IOVs to the different sub-detectors.  Currently we use
the same IOVs for BPIX,FPIX,TIB,TID,TEC which are basically determined by
movements of the pixel high-level structures that we observe with PCL and by
known magnet ramps.

Could you elaborate on how you determined the IOVs per sub-detector?

### Nazar's answer

If you mean the purely technical side, the configuration of different sets of
IOVs to different subdetectors should be achieved by extending the
`process.AlignmentProducer.RunRangeSelection` list. See the MP campaigns listed
below.

If you’re asking about the way we determined those boundaries, I think it was
done by ry running alignment with very fine time granularity (probably
run-by-run) and looking at determined alignment corrections vs time. The last
time that I’m aware of, it was done by Nastya, as you may see here:
https://indico.cern.ch/event/243933/contributions/1561044/attachments/416110/578058/Thursday_13.06.13.pdf

## On the requirement of running BP together with LA or if LA-only is enough

### Question to Nazar Bartosik

If I am not mistaken you need peak/deco data to determine BP corrections and
0T/3.8T data to determine the LA. As both of them are connected, I was wondering
if you tried to determine only the LA under the assumption of fixed BP
corrections. I am asking because we lack peak data at the moment.

### Nazar's answer

Of course you can. That’s what we did before the Backplane-calibration code was
implemented. In theory it is going to be less precise, but in practice those
backplane corrections looked strange in the end. So if you have no peak data,
you can just use the default BP corrections and let the alignment compensate for
possible BP miscalibration. Residual effect on hit positions should anyway be
much smaller compared to pure alignment without LA calibration.


## MP campaigns to look at

* mp1337 - alignment + LA
* mp1338 - alignment + LA + BP
* both are archived on `/eos/cms/store/group/alca_trackeralign/MPproduction`


## Final status report from Gregor

https://indico.cern.ch/event/688840/?showDate=all&showSession=all#35-la-calibration-in-alignment


# Automatic determination of dataset weights in MillePede

The idea is to check the track numbers after running the mille jobs and assign
default optimal weights based on the track statistics per track topology.

* requires part of the automatic validation to be run **before** the pede job
  -> easy
* use machine learning methods to learn from many combinations tracks
  -> time consuming part
  * perform study with MC
  * try different sets
    * MinBias + ZMuMu (done often during 2017)
	* MinBias + Cosmics (CRUZET/CRAFT) (likely scenario at start-up)
	* MinBias + ZMuMu + IsoMu + Cosmics (CRUZET/CRAFT) + UpsilonMuMu (EOY style)
  * try different numbers of tracks
  * try different weights
  * try different initial misalignment scenarios (but not too extreme)
* possible figures of merit:
  * fit of `c` in `delta_z = c * z` for each subdetector (each side for endcaps)
  * sine wave fit of `A` in `delta_X = A * sin(phi + C)` where `X` can be any of
    the variables in the geometry comparison plots; mainly relevant for barrel


# TODO-List for newcomers

* LSF rights for `ALCA_TRACKERALIGN` and `ALCA_MILLEPEDE` you can get them via
  the TkAl conveners
  -> also space at `/store/caf/user/${USER}`
  -> also AFS rights: `_calca_:ata` (check with command `pts m -nameorid ${USER}`)
* HTCondor rights
  -> cms-millepede
* LSF rights for `ALCA_EXPRESS` from AlCa/DB conveners
  -> cmsexpress
* the EOS group space via e-group subscription and hence from e-group admins
  ->  cms-eos-alca-trackeralign
  ->  cms-eos-alca-millepede

## Additional steps for new conveners

* Get admin rights for `ALCA_TRACKERALIGN` and `ALCA_MILLEPEDE`
  -> granted by AlCa/DB conveners or previous TkAl conveners via
     [web interface](https://lsfwebng.web.cern.ch/service-lsfweb/)
* Ask Tracker DPG conveners to be added to `cms-tracker-alignment-conveners`
  e-group
