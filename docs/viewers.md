# Viewing IMC data

During [image processing](process.md), raw IMC&trade; data files are converted into multi-channel `.tiff` or `.ome.tiff` files.
These can be visualized using common image viewers, including [ImageJ/Fiji](https://imagej.net/software/fiji/) and [QuPath](https://qupath.github.io/).
Specialized image viewers are necessary to visualize acquisitions, panoramas and segmented objects.

## MCD viewer

The [MCD viewer](https://www.fluidigm.com/products-services/software) distributed by Fluidigm&reg; for **Windows only** allows visualization of MCD&trade; files.
The user can view individual acquisitions (also referred to as regions of interest) and multiple channels. 

## histoCAT 

The [histoCAT](https://github.com/BodenmillerGroup/histoCAT)[^fn1] software offers interactive visualization of IMC data.
It requires pre-processing of raw IMC data using the IMC Segmentation Pipeline (see [image processing](process)) to generate single-channel `.tiff` files.
When supplied segmentation masks, `histoCAT` can extract object-specific features and supports clustering, dimensionality reduction and interaction testing.

!!! note ""

    Read more in the [Docs](https://github.com/BodenmillerGroup/histoCAT/releases/download/histoCAT_1.76/histoCATmanual_1.76.pdf) and in the [paper](https://www.nature.com/articles/nmeth.4391).

## histoCAT-web

A browser-based implementation of `histoCAT` is provided by [histoCAT-web](https://github.com/BodenmillerGroup/histocat-web). 
Once deployed, `histoCAT-web` reads raw IMC data files and offers an extended set of `histoCAT` functionalities.

!!! note ""

    Read more in the [Docs](https://bodenmillergroup.github.io/histocat-web/).

## napari-imc

[napari](https://napari.org/) is a fast, interactive, multi-dimensional image viewer, and the [napari-imc](https://github.com/BodenmillerGroup/napari-imc) plugin can be used to directly read raw IMC data. The `napari-imc` plugin supports visualization of panoramas and multi-channel acquisitions in the machine's coordinate system (i.e., the panoramas and acquisitions are spatially aligned with respect to each other).

<!--  Upon opening MCD files, `napari-imc` displays a graphical user interface for loading panoramas, acquisitions and channels. 
For each loaded panorama and for each combination of loaded acquisition and channel, `napari-imc` creates an image layer. 
In napari, image layers represent single-channel grayscale or color images that can be overlaid in the main panel. 
Importantly, all image layers are spatially aligned. 
Adjusting channel settings will broadcast the chosen values to the settings of all associated image layers. -->

!!! note ""

    Read more in the [paper](https://www.biorxiv.org/content/10.1101/2021.11.12.468357v1).

[^fn1]: Shapiro D. _et al._ (2017) histoCAT: analysis of cell phenotypes and interactions in multiplex image cytometry data. Nature Methods
