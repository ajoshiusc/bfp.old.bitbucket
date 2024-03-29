# BFP Workflow

## Creating Directory Structure
BFP stores data in [BIDS](http://bids.neuroimaging.io/) format. The inputs are T1 and fMRI images. They are stored in a subject subdirectory inside the study directory. T1 images are stored in anat subdirectory and fmri images in func subdirectory. There can be only one T1 image but multiple fMRI images are supported. 

## Anatomical Processing
The analtomical processing pipeline in BFP is based on [BrainSuite](http://brainsuite.org). 
* First, we resample the T1 image to 1mm cubic resolution. This is because BrainSuite tools perform optimally at that resolution. 
* Next we perform skull extraction of the T1 image. This is done using [bse](http://brainsuite.org/processing/surfaceextraction/bse/) executable in BrainSuite.
* The extracted image is then coregistered using [FLIRT](https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/FLIRT) to the BCI-DNI atlas that is included with BrainSuite. This step is performed for both fMRI and T1 images so that both are in the common space. 
* The [CSE sequence](http://brainsuite.org/processing/surfaceextraction/) in BrainSuite is then executed on the coregistered image (skull stripping is skipped). This performs tissue classification and generate inner, mid and pial cortical surface representations.
* The brain registration and labeling is performed using [SVReg](http://brainsuite.org/processing/svreg/) with BCI-DNI as the atlas.

## Functional Processing
The functional data is processed using several tools from [AFNI](https://afni.nimh.nih.gov/) and [FSL](https://fsl.fmrib.ox.ac.uk/fsl/fslwiki) as detailed below. A good reference for fMRI processing is this [wikibook](https://en.wikibooks.org/wiki/Neuroimaging_Data_Processing#Functional_MRI).
* We generate 3mm isotropic representation of BCI-DNI atlas as a standard reference. fMRI data is coregistered to this template. This steps puts subjects T1 and fMRI data in a common space.
* The fMRI data is deobliqued. This step changes some of the information inside a fmri dataset's header and is reoriented in FSL friendly space. 
* The motion correction is performed to average of time series. This is followed by skull stripping.
* Spatial smoothing is performed with Gaussian kernel and FWHM of 2mm.
* This is followed by [grand mean scaling](http://dbic.dartmouth.edu/wiki/index.php/Global_Scaling) to account for signal differences across sessions.
* A temporal band pass filtering is then applied with bandwidth of 0.009-0.1 Hz. This bandwidth is suitable for resting fMRI studies but may need adjustment for other studies.
* Detrending is performed by removing linear and quadratic trends.
* Nuisance signal regression is done using tools from FSL. First, the tissue fraction image generated by BrainSUite is coregistered to the example fMRI volume (8th volume). Using the tissue fraction image, the Global, CSF and WM average signals are regressed out from the fMRI data using [`feat_model`](https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/FEAT/UserGuide) in FSL.
* An example volume from the fMRI time series (8th volume) is coregistered to the 3mm istropic BCI-DNI atlas. This transformation is stored for use later.
* The residuals after nuisance regression are coregistered to the standard atlas using the transformation stored in previous step.

## fMRI Grayordinates
The Anatomical and Functional processing stages described above generate cortical surface representations, segmented brain volume as well as preprocessed fMRI data in a common space. 
* The data from fMRI volumes is interpolated to the subjects mid-cortical surface meshes using a linear interpolation. This data is then transferred from subject surface to BCI-DNI atlas surface using the coregistered flat maps of SVReg.
* For the volumetric grayordinates, we apply the inverse map of SVReg to map grayordinate points from BCI-DNI atlas to subject and sample the fMRI data at those coordinates using linear interpolation. 
* The surface and volume grayordinates are combined to form a vector of size 96k (32k each hemisphere + 32k subcortex).
* tNLMPdf filtering is applied on the grayordinates (both surface and volume) to generate filtered data. 

### Note on generation of grayordinates for BCI-DNI atlas
 We perform the following operations to generate Grayordinate representation of our data.
* First we identified surface and volumetric grayordinates of the BCI-DNI atlas. In order to be consistent with HCP grayordinates, we transferred HCP's grayordinates to BCI-DNI atlas in following steps.
* For generating surface grayordinates, we processed BCI-DNI atlas using FreeSurfer which generates a coregistered spherical map of the cortical surface meshes to the fsaverage atlas. The spehrical map of the grayordinates is made available in the same space by HCP [here](https://github.com/Washington-University/Pipelines/tree/master/global/templates/standard_mesh_atlases). The grayordinates are transferred to the BCI-DNI atlas using this correspondence.   
* For volumetric grayordinates of BCI-DNI atlas, we coregistered BCI-DNI atlas to BrainSuiteAtlas1 which is same as the MNI atlas, using SVReg. The grayordinate files provided by HCP make available 3D coordinates of each of the volumetric grayordinates in MNI space. We use mapping generated by SVReg to transfer these to the BCI-DNI atlas. 
