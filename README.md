# CellHorizontalProfile

## Basic concept.
- Open two images, one being "imgRed.tif" and other being "imgYellow.tif".
- Select image that has "Red" channel.
- Press "s" button to extract profile.
- All processed images will be localized in the same location as the input images.
 
## Macro code.
```java
/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*// Main profile extraction macro.
macro "Macro extract profiles. [s]"{
// Aquiring image properties.
selectWindow("imgRed.tif");
run("Duplicate...", "title=RedOrig");
selectWindow("imgYellow.tif");
run("Duplicate...", "title=YellowOrig");

selectWindow("imgYellow.tif");
run("8-bit");
selectWindow("imgRed.tif");
run("8-bit");
dirIMG = getDirectory("image");
varName = getTitle();
varNameXcut = substring(varName, 0, lengthOf(varName) - 4);
getDimensions(width, height, channels, slices, frames);
varWidth = width; varHeight = height; varChannels = channels; varSlices = slices; varFrames = frames;

// Preparing image for masking.
run("Duplicate...", "title=imgMask");
// This next line is important to set values to fit your exact image properly. In this xample images it is 5 and 255.
setThreshold(5, 255);
run("Convert to Mask");
run("Fill Holes");
run("Watershed");

// Setting measurement parameters.
run("Set Measurements...", "area mean standard modal min centroid center perimeter bounding fit shape feret's integrated median skewness kurtosis area_fraction stack display redirect=None decimal=2");

// Analyzing paricles and extracting positioning parameters from Results.
selectWindow("imgMask");
run("Set Scale...", "distance=1 known=1 unit=unit");
run("Analyze Particles...", "display exclude clear include add");
varResults = nResults;
roiManager("Show None");
roiManager("reset");

selectWindow("imgMask");
for (i = 0; i < varResults; i++) {
	varBX = getResult("BX", i);
	varBY = getResult("BY", i);
	varW = getResult("Width", i);
	varH = getResult("Height", i);
	makeLine(varBX - 2, varBY + (varH / 2), varBX + varW + 1, varBY + (varH / 2));
	roiManager("Add");
	}
run("Clear Results");
varROIs = roiManager("count");

// Iterating thru all ROIs.
for (i = 0; i <= varROIs - 1; i++){
	// Checking if both Yellow and Red profiles are the same length and have no holes across length.
	// Checking Red profile parameters.
	selectWindow("imgRed.tif");
	roiManager("Select", i);
	profile = getProfile();
	profileRed = getProfile();
	// Checking if the total holes (RT) count is equat to Start holes (RS) counts + End holes (RE) count (finding holes inside of profile).
	varHoleCounter_RT = 0;
	for (p = 0; p < profile.length; p++){
		if (profile[p] <= 0) {varHoleCounter_RT++;}
		}
	varHoleCounter_RS = 0;
	for (p = 0; p < profile.length; p++){
		if (profile[p] <= 0) {varHoleCounter_RS++;}
		else {break;}
		}
	varHoleCounter_RE = 0;
	for (p = profile.length - 1; p >= 0; p--){
		if (profile[p] <= 0) {varHoleCounter_RE++;}
		else {break;}
		}
	varIsHoleInRed = 0;
	if (varHoleCounter_RT - varHoleCounter_RS - varHoleCounter_RE > 0) {varIsHoleInRed = 1;}
	varTooLongRed = 0;
	if (profile.length >= 40) {varTooLongRed = 1;}
	
	// Checking Yellow profile parameters.
	selectWindow("imgYellow.tif");
	roiManager("Select", i);
	profile = getProfile();
	profileYellow = getProfile();
	// Checking if the total holes count is equat to Start holes counts + End holes count (finding holes inside of profile).
	varHoleCounter_YT = 0;
	for (p = 0; p < profile.length; p++){
		if (profile[p] <= 0) {varHoleCounter_YT++;}
		}
	varHoleCounter_YS = 0;
	for (p = 0; p < profile.length; p++){
		if (profile[p] <= 0) {varHoleCounter_YS++;}
		else {break;}
		}
	varHoleCounter_YE = 0;
	for (p = profile.length - 1; p >= 0; p--){
		if (profile[p] <= 0) {varHoleCounter_YE++;}
		else {break;}
		}
	varIsHoleInYellow = 0;
	if (varHoleCounter_YT - varHoleCounter_YS - varHoleCounter_YE > 0) {varIsHoleInYellow = 1;}
	varTooLongYellow = 0;
	if (profile.length >= 40) {varTooLongYellow = 1;}
	
	// Getting merged profile to estimate cell size.
	arrayProfile = newArray(profile.length);
	for (p = 0; p < profileRed.length; p++){
		arrayProfile[p] = profileRed[p] + profileYellow[p];
		}
	
	varHoleCounter_MS = 0;
	for (p = 0; p < arrayProfile.length; p++){
		if (arrayProfile[p] == 0) {varHoleCounter_MS++;}
		else {break;}
		}
	varHoleCounter_ME = 0;
	for (p = arrayProfile.length - 1; p >= 0; p--){
		if (arrayProfile[p] == 0) {varHoleCounter_ME++;}
		else {break;}
		}
	varCellWidth = (arrayProfile.length - varHoleCounter_MS - varHoleCounter_ME) - 1;
	varCellStart = varHoleCounter_MS;
	
	// Normalizing width values.
	arrayWidth = newArray(profile.length);
	for (p = 0; p < arrayProfile.length; p++){
		if (p < varCellStart) {arrayWidth[p] = 0;}
		else if (p > varCellStart + varCellWidth) {arrayWidth[p] = 0;}
		else {arrayWidth[p] = ((p - varCellStart) / varCellWidth) * 100;}
		}

	// Writing results.
	xResults = nResults;
	for (p = 0; p < profile.length; p++){
		setResult("Cell #", xResults + p, i + 1);
		setResult("Width [pix]", xResults + p, p);
		setResult("Width [norm]", xResults + p, arrayWidth[p]);
		setResult("Red [au]", xResults + p, profileRed[p]);
		setResult("Yellow [au]", xResults + p, profileYellow[p]);
		
		// If cell is too long in one channel it will have value 1, if in both channels then it will have value 2.
		setResult("CellTooLong", xResults + p, varTooLongRed + varTooLongYellow);
		// If 0 values were detected inside Red profile then the value will be 1, otherwise 0.
		setResult("HoleInRed", xResults + p, varIsHoleInRed);
		// If 0 values were detected inside Yellow profile then the value will be 1, otherwise 0.
		setResult("HoleInYellow", xResults + p, varIsHoleInYellow);
		}
	}
saveAs("Results", dirIMG + "Results.csv");

// Making overlay images and saving them.
selectWindow("RedOrig");
roiManager("Show All without labels");
run("From ROI Manager");
run("Flatten");
saveAs("Tiff", dirIMG + "imgRed_overlay.tif");
close();

selectWindow("YellowOrig");
roiManager("Show All without labels");
run("From ROI Manager");
run("Flatten");
saveAs("Tiff", dirIMG + "imgYellow_overlay.tif");
close();

// Saving Roi data to file.
roiManager("Select", 0);
run("Select All");
roiManager("Save", dirIMG + "RoiSet.zip");
}
/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*/*//
```
