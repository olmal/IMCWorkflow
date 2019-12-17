---
title: "Data import"
author: "Nils Eling"
output:
  md_document:
    variant: markdown_github
editor_options:
  chunk_output_type: console
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE, message=FALSE)
library(BiocStyle)
```

# Data import

Raw image files in .mcd format are processed using the [IMC Segementation Pipeline](https://github.com/BodenmillerGroup/ImcSegmentationPipeline).
The pipeline creates a number of folders that contain relevant files for image and single-cell data analysis.
For single-cell IMC data analysis, the cell-level intensity values per marker need to be linked to (i) cell- and image-specific metadata, (ii) marker-specific metadata and (iii) neighbourhood information.
These data are commonly stored in following files:

* __cells.csv__ (or similar): contains the cell-level information of a multitude of features. While the majority of these features should be regarded as cell-level metadata (e.g. shape and position), usually the _MeanIntensity_ and __MeanIntensityCorrected__ columns contain the raw and spill-over-corrected counts per cell and per channel.
* __Image.csv__: contains image-level metadata. These include the image name, number of detected cells, and most importantly the scaling factor, with which the counts need to be multiplied to account for the image encoding (e.g. 16-bit images: scaling factor = 2^{16}-1 = 65535).
* __Object Relationships.csv__: contains a table of relationships between neighbouring cells.
* __Panel.csv__: to associate the channel names with the metal-labelled antibodies, the user needs to specify a file that conatins the link between (i) antibody, (ii) metal-tag and (iii) channel name. 

__TODO: more information on how to generate these files using CellProfiler.__ 

The files __cells.csv__, __Image.csv__ and __Object relationships.csv__ can be found in the __cpout__ folder created by the __IMC Segmentation Pipeline__.
However, the location of the __Panel.csv__ file is not controlled and needs to specified by the user.

```{r read-in-data}
cells <- read.csv(file = "data/extdata/cells.csv", stringsAsFactors = FALSE)

image <- read.csv(file = "data/extdata/image.csv", stringsAsFactors = FALSE)

relationships <- read.csv(file = "data/extdata/Object relationships.csv", 
                            stringsAsFactors = FALSE)

panel <- read.csv(file = "data/extdata/panel.csv", stringsAsFactors = FALSE)
```

## Data cleaning

The __cell.csv__ contains `r ncol(cells)` cell-level features.
These include cell-specific intensity counts per marker as well as positional information, object and image identifiers.
These are crucial to link cells to individual images (and image-level metadata) as well as to segmented objects on images.
The `colnames` of the __cells.csv__ file indicates the measured features. 
All channel (marker)-specific intensity features are stored in the format `..._cX`, where X indicates the channel number.   
An overview on which features were acquired can be obtained after excluding the channel ID.

```{r cell-features}
unique( sub("_c[0-9]*$", "", colnames(cells)) )
```

### Selecting cell-specific intensities

A number of different area, intensity, location and neighbourhood features were selected for export from `CellProfiler`.
For more details on these features, please refer to the manual ([CellProfiler Measurement](https://cellprofiler-manual.s3.amazonaws.com/CellProfiler-3.1.9/modules/measurement.html#)).
To obtain cell-specific intesities for each marker, a number of different measurements are available:

* __IntegratedIntensity__: Sum of pixel intensities across all pixels of each cell
* __MeanIntensity__: Mean pixel intensity across all pixels of each cell
* __MedianIntensity__: Median pixel intensity across all pixels of each cell

Furthermore, these measures can be calculated on different image stacks:

* __FullStack__: Contains the raw pixel intensities 
* __FullStackFiltered__:  Contains the pixel intensities after applying a selective median filter to remove 'hot pixels' (unusually bright pixels)
* __ProbabStack__: Contains the probability for each pixel to be associated with a certain class selected during pixel classification (__Refs!__)

In addition, intensities can corrected for spill-over between channels (__Cell Systems ref__).
This is either done on the pixel- or cell-level intensities.
These cell-level corrected features are either stored as __MeanIntensityCorrectedLS__ using a least-squares regression for correction or _MeanIntensityCorrected_ using a non-negative least-squares regression for correction (__Refs!__)
For our purposes, we will work on the spillover corrected counts averaged across each cell after removing 'hot pixels' (unusually bright pixels). 
To avoid negative values, we will select the non-negative least-squares corrected counts, which can be accessed via:

```{r select-counts}
cur_counts <- cells[,grepl("Intensity_MeanIntensityCorrected_FullStackFiltered", colnames(cells))]
```

By default, CellProfiler scales all pixel intensities between 0 and 1.
This is done by dividing each count by the maximum possible intensity value (see [MeasureObjectIntensity](https://cellprofiler-manual.s3.amazonaws.com/CellProfiler-3.0.0/modules/measurement.html#measureobjectintensity) for more info).
In the case of 16-bit encoding (where 0 is a valid intensity), this scaling value is `2^16-1 = 65535`.
Therefore, to obtain raw cell-specific intensities, mean intensites are scaled by this value. 
CellProfiler by defaults stores the scaling factor in the image metadata file:

```{r scale-counts-1}
image$Scaling_FullStack
```

Intensities are scaled as follows:

```{r scale-counts-2}
cur_counts <- cur_counts * image$Scaling_FullStack[1]
```

__Please make sure to scale images accordingly if you use different encodings.__

### Cell-specific metadata

The `SingleCellExperiment` allows storing cell/image- and marker-specific metadata.
On the per-cell-level, metadata include:

* the cell's number (identifier)
* the cell's location
* the cell's shape features

These information are stored in a [DataFrame](https://bioconductor.org/packages/release/bioc/manuals/S4Vectors/man/S4Vectors.pdf) object of the `S4Vectors` package and can be obtained from the __cell.csv__ file.

```{r cell-metadata}
library(S4Vectors)
cell_meta <- DataFrame(CellNumber = cells$ObjectNumber,
                       Center_X = cells$Location_Center_X,
                       Center_Y = cells$Location_Center_Y,
                       Area = cells$AreaShape_Area,
                       MajorAxisLength = cells$AreaShape_MajorAxisLength,
                       MinorAxisLength = cells$AreaShape_MinorAxisLength)
```

A number of cell-specific metadata can be obtained using CellProfiler.
The [CellProfiler manual](https://cellprofiler-manual.s3.amazonaws.com/CellProfiler-3.1.9/modules/measurement.html#) provides more information on further features.

### Image-specific metadata

Image-specific metadata can be stored together with cell-specific metadata.
These will be added to the cell-specific metadata `DataFrame`.
Alternatively, the [CATALYST](https://bioconductor.org/packages/release/bioc/html/CATALYST.html) package supports storing the image-specific metadata in an `experiment_info` entry in the `metadata(x)` slot of the `SingleCellExperiment` object.
Cells and images are linked via a `sample_id` entry in `colData(x)` and `metadata(x)$experiment_info`.

Image-specific metadata are stored in the __Image.csv__ file produced by CellProfiler (__Ref__).
More specifically, this file contains image- and acquisition-specific meatdata.
Acquisition metadata can be idenfified by the `Metadata_` entries:

```{r show-ac-metadata}
colnames(image)[grepl("Metadata_", colnames(image))]
```

Specifically, the `Metadata_acname` and `Metadata_roiid` contain information that need to be stored to uniquely identify cells from different images.
The `cells$ImageNumber` entry associates each cell to its corresponding image in the __Image.csv__ file.
That allows linking cells to image-specific metadata entries.
Here, the image number, batch ID and sample ID will be stored in the cell-specific metadata object.
For easier string subsetting, the [stringr](https://cran.r-project.org/web/packages/stringr/vignettes/stringr.html) package can be used.

```{r store-image-metadata}
library(stringr)

image$Metadata_acname
image$Metadata_roiid

# Store image number in cell-specific metadata object
cell_meta$ImageNumber <- cells$ImageNumber

# Split acquisition information
ac_info <- str_split(image$Metadata_acname, "_", simplify = TRUE)
cell_meta$BatchId <- ac_info[cell_meta$ImageNumber,2]
cell_meta$SampleId <- ac_info[cell_meta$ImageNumber,3]
cell_meta$ROI <- image$Metadata_roiid[cell_meta$ImageNumber]
```

It is recommended to store unique cell IDs as `rownames` of the cell-specific metadata object
This will be crucial when merging `SingleCellExperiment` objects in cases where multiple batches were at first analysed independently.
For this example dataset, the sample ID and ROI uniquely defines an image:

```{r set-rownames-cells}
rownames(cell_meta) <- paste(cell_meta$SampleId, cell_meta$ROI, cell_meta$CellNumber, sep = "_")
```

### Feature-specific metadata

The __Panel.csv__ contains all informative metadata describing the markers (features) measured in the experiment.

```{r show-panel}
library(DT)
DT::datatable(panel)
```

However, this file is manually created and therefore error-prone.
During pre-processing using the [IMC Segementation Pipeline](https://github.com/BodenmillerGroup/ImcSegmentationPipeline), image-specific metadata files are created that link the channel number with the measured isotopes.
This information can be found in the `_full.csv` files located in the `tiffs` output folder produced by the IMC segementation pipeline.
For purposes of this workflow, the channel information is saved in the __extdata__ folder.
If all images have been processed simultaneously, the ordering of channels is consistent for all images.

__TODO: Make sure that this is documented somewhere__

```{r channel-mass}
channel_mass <- read.csv("data/extdata/IMMUcan_Batch20191023_10032143-THOR-VAR-TIS-UNST-03_s0_p5_r1_a1_ac_full.csv", 
                          header = FALSE)
channel_mass
```

Here, the rownames indicate the channel number and the first column stores the isotope mass (e.g. _In113_ for _Indium_ with the atomic mass number _113_).
It is now crucial to re-order the panel information based on the order of the measured channels. 

```{r reorder-panel}
# Order panel by isotope
panel <- panel[match(channel_mass[,1], panel$Metal.Tag),]
```

The `panel$Target` entry contains the correct annotation of proteins measured in the experiment.
These should be used as `rownames` of the `panel` data frame and later on the `SingleCellexperiment` object.

```{r set-rownames-panel}
# Use clean target as rownames
rownames(panel) <- panel$Target
```

By default, the channel measures are not correctly ordered in the __cells.csv__ file.

```{r channel-order}
colnames(cur_counts)
```

It is therefore crucial to re-order the column vectors to match the panel information

```{r reorder-counts}
# Get channel number
channelNumber <- as.numeric(sub("^.*_c", "", colnames(cur_counts)))

# Order counts based on channel number
cur_counts <- cur_counts[,order(channelNumber, decreasing = FALSE)] 
```

These steps now produced an object that stores the cell- and marker-specific intensities, an object that stores the cell-specific metadata and an object that stores the marker-specific metadata.

### Spatial data

The [IMC Segementation Pipeline](https://github.com/BodenmillerGroup/ImcSegmentationPipeline) allows user to store information of neighbouring cells (based on a user defined distance threshold, __REF!__).
__TODO: more explanation on where this is comming from__.
The computationally most efficent way to store this neighbourhood information is in form of a `graph` object.
Here, we are using the [igraph](https://igraph.org/) package to construct the graph based on __Object relationship.csv__ file.
This `igraph` object can be stored in the `metadata(x)` slot of the `SingleCellExperiment` object, which does not allow correct and consistent sub-setting.
The graph is constructed using the unique cell IDs rather than `ImageNumber_ObjectNumber` to allow consitent analysis across sample batches.

```{r igraph-object, message=FALSE}
library(igraph)
# Construct neighbour data.frame
# First in the ImageNumber_ObjectNumber format
cur_df <- data.frame(CellID_1 = paste0(relationships$First.Image.Number, 
                                       "_", relationships$First.Object.Number),
                     CellID_2 = paste0(relationships$Second.Image.Number, 
                                       "_", relationships$Second.Object.Number))

# Create simple cell IDs
cellID <- paste0(cell_meta$ImageNumber, "_", cell_meta$CellNumber)

# Change cell IDs
cur_df$CellID_1 <- rownames(cell_meta)[match(cur_df$CellID_1, cellID)]
cur_df$CellID_2 <- rownames(cell_meta)[match(cur_df$CellID_2, cellID)]

# Build graph
g <- graph_from_data_frame(cur_df)
g
```

# Create the SingleCellExperiment object

After collecting all relevant intensities, metadata and cell relationships, all data can be stored in a `SingleCellExperiment` object.
As of `Bioconductor` convention, cells are stored in the columns and features are stored in the rows.
In the first step, the `SingleCellExperiment` object is created based on the cell-specific marker intensities and `rownames` and `colnames` are propagated from the metadata to the `SingleCellExperiment` object.

```{r create-SCE}
library(SingleCellExperiment)
# Create SCE object
sce <- SingleCellExperiment(assays = list(counts = t(cur_counts)))

# Set marker name as rownames and cellID as colnames
rownames(sce) <- rownames(panel)
colnames(sce) <- rownames(cell_meta)
```

Next, the cell-level metadata are stored in the `colData(x)` slot, the panel information in the `rowData(x)` slot and the graph object in the `metadata(x)` slot.

```{r store-metadata}
colData(sce) <- cell_meta
rowData(sce) <- panel
metadata(sce) <- list(graph = g)
sce
```

# Save SCE object

The `SingleCellexperiment` can now be saved to disc and easily accessed using the `saveRDS` function.

```{r save-RDS}
saveRDS(sce, "output/sce.rds")
```


