Respiration Artifact Filter
=========================================

I. How to Run the Program

Fully extract the entire folder. Do not copy or run the EXE file by itself.

Double-click Respiration_Artifact_Filter.exe.

Select the corresponding original multi-frame grayscale TIFF file.

Select the previously generated SSIM Map TIFF file.

Click Analyze / Load Cache.

After the analysis is complete, the program will display all frames marked for removal under the Standard mode by default.

Based on your review, select Conservative, Standard, or Sensitive.

Click Generate AF TIFF.

The files must satisfy the following correspondence:

The original TIFF contains N frames.

The SSIM Map must contain N−1 frames.

SSIM Map frame i corresponds to frames i and i+1 of the original TIFF.



II. Detection Modes

Conservative: Minimizes the risk of removing genuine intestinal motility, but may miss mild respiratory artifacts.

Standard: Recommended by default. Balances false removal and missed detection.

Sensitive: Detects weaker respiratory artifacts, but may remove a small number of genuine motion frames.

Switching between modes does not reload the TIFF files or repeat the analysis. It only updates the list of frames marked for removal in the log window.



III. Analysis Log

After the analysis is complete, the log window in the main interface will display:

The currently selected mode;

The number and percentage of frames marked for removal;

The complete frame ranges marked for removal;

The full list of SSIM frame numbers marked for removal, numbered from 1;

The number of frames marked for removal under each of the three modes.

In the log, SSIM frame i corresponds to frames i and i+1 of the original TIFF.



IV. Analysis Outputs and Cache

After analysis, the program will generate the following files in the folder containing the SSIM Map:

SourceSSIMFileName_Respiration_Analysis_Cache.npz

SourceSSIMFileName_Respiration_Analysis.csv

SourceSSIMFileName_Respiration_Analysis_Preview.svg

The cache file prevents unnecessary repeated calculations. The CSV file records frame-by-frame metrics and the classification results for all three modes. The SVG file shows the respiratory-artifact score curve.



V. AF Output

The currently selected mode determines which frames are removed. The output filename is always:

SourceSSIMFileName_AF.tif

If a file with the same name already exists, the program will ask whether it should be overwritten.

The output is saved as a 32-bit floating-point, ImageJ-compatible TIFF stack.



VI. Automatic Detection Strategy

The program performs an integrated analysis of:

Global displacement between consecutive frames in the original TIFF;

Motion-direction consistency across a 4 × 4 grid;

The proportion of image blocks showing motion;

The mean and lower-percentile values of each SSIM Map;

The area of low-SSIM regions relative to the stable baseline of the same video.

The low-SSIM threshold is estimated automatically for each video. The program also searches forward and backward from clearly detected respiratory peaks to determine the boundaries of continuous respiratory events automatically.



VII. Important Notes

Physically removing frames compresses the time axis. If strict analysis of the original time intervals is required, the original SSIM frame numbers recorded in the analysis CSV should be used.

The algorithm should continue to be validated using additional real datasets. Before formal statistical analysis, it is recommended to spot-check the original video, the SSIM Map, and the frames marked for removal in the log.



VIII. Supported Formats and Systems

64-bit Windows

Original TIFF: 8-bit or 16-bit unsigned, single-channel, multi-frame grayscale TIFF

SSIM Map: 32-bit or 64-bit floating-point, single-channel, multi-frame TIFF

The original TIFF and SSIM Map must have the same image width and height

The output can be opened as a 32-bit stack in ImageJ/Fiji



IX. Runtime Environment

This program does not install Python, Conda, or any system-level dependencies. It does not modify the system PATH, the Windows registry, or any existing scientific computing environment.

The program folder contains an isolated CPython 3.11 runtime together with NumPy and tifffile.

Respiration_Artifact_Filter.exe contains the original binary content of pythonw.exe from the CPython embeddable package; only the filename has been changed.

The actual program logic is stored in respiration_filter_app.py, which can be opened and inspected directly.



Do not delete or move any of the following files or folders:

python311.dll

python311.zip

python311._pth

the Lib folder

sitecustomize.py

respiration_filter_app.py
