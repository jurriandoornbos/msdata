<div align="center">
  <p>
    <a href="https://icaerus.eu" target="_blank">
      <img width="50%" src="https://raw.githubusercontent.com/ICAERUS-EU/.github/refs/heads/main/profile/ICAERUS_transparent.png"></a>
    <h3 align="center"> Multispectral UAV Image calibration and alignment</h3>
    
   <p align="center">
    Python-based calibration procedures for DJI Mavic 3M, DJI Phantom 4 Multispectral, MicaSense Altum (-pt), RedEdge and Parrot Sequoia.
    <br/>
    <br/>
    <a href="https://github.com/icaerus-eu/msdata/wiki"><strong>Explore the wiki »</strong></a>
    <br/>
    <br/>
    <a href="https://github.com/icaerus-eu/msdata/issues">Report Bug</a>
    -
    <a href="https://github.com/icaerus-eu/msdata/issues">Request Feature</a>
  </p>
</p>
</div>

![Downloads](https://img.shields.io/github/downloads/icaerus-eu/repo/msdata/total) ![Contributors](https://img.shields.io/github/contributors/icaerus-eu/msdata?color=dark-green) ![Forks](https://img.shields.io/github/forks/icaerus-eu/msdata?style=social) ![Stargazers](https://img.shields.io/github/stars/icaerus-eu/msdata?style=social) ![Issues](https://img.shields.io/github/issues/icaerus-eu/msdata) ![License](https://img.shields.io/github/license/icaerus-eu/msdata) 

## Table Of Contents
- [Summary](#summary)
- [Installation](#installation)
- [Usage](#usage)
- [License](#license)
- [Authors](#authors)
- [Acknowledgments](#acknowledgments)


## Summary
Calibration procedures for various Multispectral sensors and UAVs. Used to create the large, open, pretraining dataset MSUAV.

Calibration for multispectral sensors in this repository covers `alignment` and `reflectance` calibration. Assuming RAW tiffs directly captured in the flight, all the necessary parameters should be in the exif tags in the raw .tif files.

## Installation
First, install [Conda](https://docs.conda.io/projects/conda/en/latest/user-guide/install/index.html). Then create an environment called `msdata` like below:

```bash
conda create --name msdata jupyterlab opencv numpy pandas rioxarray pillow piexif matplotlib tifffile
conda activate msdata
```

## Usage
There are some general notebooks going over calibration options/procedures for various sensors, based on snippets and code online available.

These are:
* `notebooks/mavic_3m.ipynb`
* `notebooks/phantom_4m.ipynb`
* `notebooks/altum.ipynb`
* `notebooks/rededge.ipynb`
* `notebooks/sequoia.ipynb`
For RedEdge, Altum and Sequoia additional panel image calibration is also included.

Then specifically for the included datasets according to the [paper](datainbrief.com), specific notebooks are available.

## License
Licensed under MIT License, see LICENSE file.

## Authors
* **Jurrian Doornbos** - *Wageningen University* - [Jurrian Doornbos](https://github.com/jurriandoornbos)

## Acknowledgments
This project is funded by the European Union, grant ID 101060643.

<img src="https://rea.ec.europa.eu/sites/default/files/styles/oe_theme_medium_no_crop/public/2021-04/EN-Funded%20by%20the%20EU-POS.jpg" alt="https://cordis.europa.eu/project/id/101060643" width="200"/>
    

# Explanation of alignment and radiometric calibration


## **1. Image Alignment Using ORB Keypoints and Affine Transformation**

Image alignment is necessary when combining different spectral bands that may have slight offsets due to sensor misalignment, drone movement, or differences in capture times. The alignment process involves detecting keypoints, matching them between images, and applying an affine transformation to align the images spatially.

### **Step 1: Keypoint Detection**
The ORB (Oriented FAST and Rotated BRIEF) algorithm is used to detect keypoints and compute descriptors for the images.

$$
K_1, D_1 = ORB(I_1) \quad \text{and} \quad K_2, D_2 = ORB(I_2)

$$

Where:
- \( K_1, K_2 \) are the sets of detected keypoints.
- \( D_1, D_2 \) are the corresponding descriptors.

### **Step 2: Descriptor Matching**
Using a FLANN-based matcher, the descriptors from the two images are compared to find corresponding keypoints. The ratio test filters matches to retain only those that meet a threshold for accuracy:

$$
\text{Good Matches} = \{m \mid m.\text{distance} < 0.8 \times n.\text{distance}\}
$$

Where \( m \) and \( n \) are pairs of matching descriptors.

### **Step 3: Affine Transformation Calculation**
The matched keypoints are used to calculate an affine transformation matrix \( A \), which accounts for translation, scaling, rotation, and slight shearing:

$$
A, \_ = \text{estimateAffinePartial2D}(\text{dst\_pts}, \text{src\_pts}, \text{method=RANSAC})
$$

Where:
- \( A \) is the affine transformation matrix.
- \( \text{dst\_pts} \) are the points from the target image.
- \( \text{src\_pts} \) are the corresponding points from the reference image.

### **Step 4: Applying the Affine Transformation**
The target image is aligned to the reference image using the affine matrix:

$$
I_{\text{aligned}} = \text{warpAffine}(I_2, A, \text{size}(I_1))
$$

Where:
- \( I_{\text{aligned}} \) is the aligned version of the target image.
- \( I_2 \) is the original target image.
- \( I_1 \) is the reference image.

This ensures that each pixel in the aligned image corresponds spatially to the same location as in the reference image.

---

## **2. Reflectance Calculation**

The reflectance calculation converts the corrected pixel values from the NIR image into reflectance values, which represent the proportion of incident light reflected by the surface.

### **Step 1: Normalization of Raw Pixel Values**
The raw pixel values are normalized to correct for sensor-specific properties and capture settings. The normalization formula is:

$$
NIR_{\text{norm}} = \left(\frac{NIR_{\text{raw}}}{65535} - \frac{BL}{65535}\right) \times \frac{10^6}{\text{Gain} \times t_{\text{exp}}}
$$

Where:
- \( NIR_{\text{raw}} \) is the raw pixel value.
- \( BL \) is the black level correction (fixed at 4096).
- Gain is the sensor gain extracted from metadata.
- \( t_{\text{exp}} \) is the exposure time in seconds.

### **Step 2: Vignetting Correction**
Vignetting causes a reduction in image brightness toward the edges of the image. The correction factor is computed as a polynomial function of the radial distance from the optical center:

$$
V(r) = \sum_{i=0}^{n} c_i \cdot r^i + 1
$$

Where:
- \( r \) is the radial distance from the optical center, calculated as:
  \[
  r = \sqrt{(x - C_x)^2 + (y - C_y)^2}
  \]
- \( C_x, C_y \) are the coordinates of the optical center.
- \( c_i \) are the vignetting coefficients.

The normalized NIR image is corrected for vignetting as follows:

\[
NIR_{\text{corrected}} = NIR_{\text{norm}} \times V(r)
\]

### **Step 3: Lens Distortion Correction**
Lens distortion is corrected using the camera matrix \( K \) and distortion coefficients \( D \):

$$
K = \begin{bmatrix} f_x & 0 & C_x + c_x \\
0 & f_y & C_y + c_y \\
0 & 0 & 1 \end{bmatrix}
$$

Where:
- \( f_x, f_y \) are the focal lengths.
- \( C_x, C_y \) are the calibrated optical center coordinates.
- \( c_x, c_y \) are offsets from the metadata.

The distortion coefficients \( D = [k_1, k_2, p_1, p_2, k_3] \) account for both radial and tangential distortions.

The corrected image is obtained by applying the undistortion:

$$
NIR_{\text{undistorted}} = \text{undistort}(NIR_{\text{corrected}}, K, D)
$$

### **Step 4: Reflectance Calculation**
The final step is to calculate the reflectance values by dividing the corrected pixel values by the irradiance:

$$
R = \frac{NIR_{\text{undistorted}}}{E}
$$

Where:
- \( R \) is the reflectance value.
- \( E \) is the irradiance, calculated as the product of the NIR light sensor reading and a calibration factor:
$$
  E = NIR_{LS} \times p_{LSNIR}
$$

The result is a radiometrically corrected image that accurately represents the reflectance properties of the observed surface.

