## MINIMA
Code for image and data analysis used in MSCA MINIMA project. 
## License
This work is licensed under a [Creative Commons Attribution 4.0 International License](https://creativecommons.org/licenses/by/4.0/).

### Data analysis procedure 
**Original data:** Bacteria single cells grow into colonies imaged by IX83

**Data structure:** date-AgarConc-ana/ID(single colony)/
* ID-c(ontrast enhanced)
* ID-BG
* ID-thre >> ID-area-xy.csv
* **ID-BG_Simple Segmentation** >> ID-tracks-cv2.csv 
  *  ID-elg-cv2.csv
  *  ID-offsprings.json
  *  ID-corr_elg_dist.csv
  *  trackmate folder for files before -tracks-cv2.csv
  *  plots: ana-plot/, plot-check-cv2/, plot-videos-cv2/, ana-correlation/
* 1stAna/: for previous analysis from self-written tracking script
* 2ndAna/: test analysis for accelaration etc.



#### Pre-processing with Fiji-ImageJ

1. Import from .vsi
2. If necessary, align images: Fiji Registration -- Linear stack alignment with SIFT
3. Crop ROI, each image stack record one cell to colony, Adjust contrast and save as -c 
_Note: pick the well-resolved, in focus colonies_ 
4. Run Fiji-Marco to: 
   * substrate background, "Rolling ball radius" 50, "slicing paraboloid". _if traces left outside of colonies, reduce 50 to 20_
   * Threshold, calculate for each slice (may reduce to include slightly larger area), -thre, save
_Fiji script doesn't work on generating mask, thresholding too noisy_
_may be better if better illuminated/exposed_
5. Fiji
   * -c invert, "AND" -thre = -BG, save (only keep colony part as ROI, background black, colony light color)
   * Run thresholding again on -BG, fix threshold value 80, new -thre

**Output:** "colony_index/"
1. **"colony_index-c.tif"**: colony development, aligned, cut ROI, contrast enhanced.
2. **"colony_index-BG.tif"**: inverted "colony_index-c", pixel=0 if outside of colony contour.
3. **"colony_index-thre.tif"**:"colony_index-BG" set threshold=80 (in 250605-600 set to 20). Keep frame up to MTMT. _if not MTMT, see README_
   * white pixels => colony area
   * Colony density = colony area/contour area
   * Last frame => t_c
   * Contour area on the last frame => A_c


#### Pixel classification with ilastik
0. The contrast between cells should be kept as much as possible, hence **"Substrate background" is not used as pre-processing, only increase contrast and -BG by cutting contour outline**
1. Build a model file with **1st frame+1st division+last key frame** of an image stack, train with this model file.
2. Load data in tyx (aaa-BG). 
3. Feature selection: check if difference can be seen under corresponding feature filter, if not, remove the feature. Usually all features from 0.5 to 5.
4. Train pixel classification, make sure cells are separated. Add training if necessary.
5. If difficult to segment, keep frames until G2/3 (16cells) _6% agarose_
5. **Post-processing**: Labeled image to 0/255 binary, manually correction in Fiji, fill hole, erode+dilate. _iterate after see TrackMate tracks_

**Output:**"colony_index/"
1. **"colony_index-BG_Simple Segmentation.tiff"**: raw segmentation results
2. **same name**: manually corrected segmentation results until frame #71
3. For a few selected, correct frames till after MTMT for elgRate map

_Note 20250612:_ f71 is still in the "lag phase", elgrt averaged from f10-71 cannot be used for argument. Best is the whole curve of local_growth_rate to time.


#### Colony Analysis in Python
0. **"Area-analysis.ipynb"**
1. Colony-level analysis with **"colony_index-thre.tif"**:
   * white pixels => colony area
   * Colony density = colony area/contour area
   * Last frame => t_c
   * Contour area on the last frame => A_c

#### Tracking / Cell-level analysis
* _Todo: MorpholibJ geodesic diameter instead of the length_
* _btrack crashes under both Python 3.11 and 3.12_
* _TrackPy give only separated trajs, cannot find division, could be nice for Brownian tracking_
* _**TrackMate overlap tracker + manual correction**_

0. **Input**: "colony_index-BG_Simple Segmentation.tiff" post-processed (manually corrected, fill hole, erode+dilate) _iterate after see TrackMate tracks_
1. **"segmentation-analysis"** for contour analysis
Read in needed frames (most 0-70, ElgRateMap all ) in **"colony_index-BG_Simple Segmentation.tiff"**. It should be in 0/255.
   * find 0-level contours, record center coords, bounding rectangle (length, width, angle), may fit ellipse. \
   * **Output DataFrame '-spots_cv2.csv'**: [frameIdx]: nparray # cellIdx, cntx, cnty, 3rect0, 4rect1, 5area, 6aspect ratio(AR), 7length, 8angle
2. TrackMate
   * Mask detector "Simplify contours"
   * Overlap tracker (minIoU 0.05, scale factor 1)
   * if needed, manual link in Track Scheme
   * "Save" output a xml file that can be load back by TrackMate
   * Output1: '-BG_spots.csv' [LABEL,ID,TRACK_ID,QUALITY,POSITION_X,POSITION_Y,POSITION_Z,POSITION_T,FRAME,RADIUS,VISIBILITY,MANUAL_SPOT_COLOR,MEAN_INTENSITY_CH1,MEDIAN_INTENSITY_CH1,MIN_INTENSITY_CH1,MAX_INTENSITY_CH1,TOTAL_INTENSITY_CH1,STD_INTENSITY_CH1,EXTRACK_P_STUCK,EXTRACK_P_DIFFUSIVE,CONTRAST_CH1,SNR_CH1,ELLIPSE_X0,ELLIPSE_Y0,ELLIPSE_MAJOR,ELLIPSE_MINOR,ELLIPSE_THETA,ELLIPSE_ASPECTRATIO,AREA,PERIMETER,CIRCULARITY,SOLIDITY,SHAPE_INDEX] use _[ID, POSITION_X,POSITION_Y, FRAME, ELLIPSE_MAJOR,ELLIPSE_THETA, SOLIDITY]_
   * Output2: '-BG_edges.csv' [LABEL,TRACK_ID,SPOT_SOURCE_ID,SPOT_TARGET_ID,LINK_COST,DIRECTIONAL_CHANGE_RATE,SPEED,DISPLACEMENT,EDGE_TIME,EDGE_X_LOCATION,EDGE_Y_LOCATION,EDGE_Z_LOCATION,MANUAL_EDGE_COLOR] use _[SPOT_SOURCE_ID,SPOT_TARGET_ID,EDGE_TIME]_

3. Python track reconstruction (**"segmentation-analysis"**)
   * Map **'_spots.csv'** with DataFrame **'-spots_cv2.csv'** output **-spots_cv2_tmmap.csv**
   * Reconstruct cell tracks from **'_spots_cv2_tmmap.csv'** and **'_edges.csv'**  ["cell_ID", "cell_parent_ID", "cell_daughter1_ID", "cell_daughter2_ID", "spot_ID", "frame_ID", "spot_x", "spot_y", "generation"] output **'-tracks-cv2.csv'**. _somehow there are always spot not found?_
   * For sanity check, use **RatePlot** plot cell_ID onto tiff(**"plot-videos-cv2/-cellID.tif"**) 
   * 
4. Correction on **'-tracks-cv2.csv'** 
   * check **"-cellID.tif"** , if some cells are not segmented, correct on **"BG_Simple Segmentation.tiff"**, re-run TrackMate + cv2
   * check the sanity check plots of each cell length-time from elongation rate calculation.
   * correct direct on **'-tracks-cv2.csv'**:
     * curved cells or abnormal cell length (segmentation error etc.) change solidity to 0.7 _will be excluded in elgRate fitting_ 
     * correct tracks due to mis-seg or lost daughter cells, keep full cell cycles [2025/06/24], corrections documented in **"README.txt"**, original saved as "**-tracks-cv2-raw.csv**"
 
5. Calculate elongation rate of each cell/ track (**growth_elongation_rates**)
   * input **'-tracks-cv2.csv'**, output **"-elg-cv2.csv"** Using bounding box length or ellipse long axis. [20250625 use bounding box length] _ellipse fitting is not accurate for long cells_
   * Filters:
     1. cell contours solidity>0.7 (exclude curved cells)
     2. After 1st filter, track length > 4

   * Calculate: 
     * **2ndAna**: cell_ID, generation, time_start, mean_solidity, Lb, Ld, elgRate_b, elgRate_start_b, elgRate_e, elgRate_start_e, division_time 
     * **20250625**: cell_ID, generation, time_start,Lb, Ld, elgRate_b, division_time
     * fit exponential to whole "elgRate"
     * first linear to first 5 "elgRate_start". [20250625 dropped]
     * **time_start**: time at birth, (time at 1st frame if "cell_parent_ID" not Null)
     * **Lb**: length right after division (length at 1st frame if "cell_parent_ID" not Null)
     * **Ld**: length before next division (length at last frame if "cell_daughter1_ID" not Null)
     * **division_time**: time span of from previous division to next division (time span of the track is both "cell_parent_ID" and "cell_daughter1_ID" not Null)
   * **Sanity check**:
     * print the cell_ID of outliers abs(x-x0)>3std
     * check the tracks and fit plot.
   * Plot colordots to tiff with **RatePlot**
   * Plot ['elgRate_b', 'Lb', 'division_time', 'Ld', 'kd', 'ketd', 'deltaL'] to ['generation', time_bin] **Statistics_of_elgRate_150**
 
6. Position analysis(**Rate-position correlation**)
   * Find out the distance of each cell to center, frame 60 +=10 till max, output **'-corr_elg_dist.csv'**
   * Analysis elgRate/Lb/division_time correlation to distance, plot scatter plot and calculate Pearson correlation.
 
7. Lineage analysis(**lineage-sort**)


8. Change of elongation rate (**Rate_change**, **growth_elongation_rates**) _need revision_
   * real-time elgRate, still need revised
   * accelation from real-time elgRate
   * Elongation rate difference from parent to children 

