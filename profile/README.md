# Repositories for Processing CT Data

Here I will briefly describe the repositories purpose and structure.
I will split description into two parts: the pipeline part will describe repositories that are part of the large pipeline aimed to end-to-end processing and the collection part will be aimed at describing off-the-pipeline repositories with small solutions.

## The Pipeline

First of all, short intro to the idea of the pipeline.
We frequently have to batch process loads of the datasets in a similar manner.
From contrast adaptation and datatype conversion all way down to segmentation and measurements.
The pipeline aimed to solve all of those steps.
Theoretically, pipeline is coded in a way, that allows to unite all steps in some kind of a queue.
However, queueing was never implemented.
Therefore up to now, each step is a separate repository and task, each batch processing the whole set of files, before the next step starts.
One should expect that each step has two ultimate parts of configuration: input (files list, folder with files, wildcarded path or something like that), output (folder, prefix to add to processed files, or in a special case -- database address).

The pipeline is not very much suited for initial experiment with super-novel ideas of the training approaches.
The primary target is to automatize and make configurable the boring part: pre-process/segment/clean/save and so on.
However, everything that experiments with changing only architecture, or singular aspects of training (e.g., novel learning rate schedule), can be easily implemented in this pipeline.
I will now describe steps and give links to repositories.

### [Image Pre-Processing](https://github.com/DL4XRayTomoImaging-KIT/BinScale3D)
The repository includes contrast adaptation and color-space binning (e.g., converting to 8-bit with removing outliers), scaling (e.g., downscaling the whole file 2 times), and automated cropping (based on either pixel color or local entropy).

### [DL Training and Inferece](https://github.com/DL4XRayTomoImaging-KIT/training-repo)
This one includes routines to train a model on a small dataset, and roll out the inference on a large set of files.
Good examples are [training to segment heart and kidney of Medaka](https://github.com/DL4XRayTomoImaging-KIT/training-repo/tree/medaka-brain_heartkidney) and [training to denoise spectral- or time-resolved tomography](https://github.com/DL4XRayTomoImaging-KIT/training-repo/tree/noise2noise).
They both use the same codebase (well, mainly, dataloaders are different), but configures for training are rather different.

### [Segmentation Post-Processing](https://github.com/DL4XRayTomoImaging-KIT/post-proceessing-repo)
Raw-ish repository, where the post-processing of the segmentations produced by a NN happens.
The idea is to drop all false positives by defining largest connected areas (first roughly as a bounding box, and then more precisely) and add some binary morphological operations on top of that (e.g., closing to remove small spongy gaps).
Previously this was part of the [measuring step](https://github.com/DL4XRayTomoImaging-KIT/measuring-repo) and can be still called from there.

### [Morphological Measurements](https://github.com/DL4XRayTomoImaging-KIT/measuring-repo)
This repository is for final steps of analysis.
For each class of the segmentation of a given volume it can measure tens of the parameters.
Starting from simple shape related ones (volume, convex volume) down to complex and color related ones (standard deviation of the color on the thin volume surroinding boundary of the segmented area).
A somewhat good example of the configuration is [this config for measurement of brain segmentation](https://github.com/DL4XRayTomoImaging-KIT/measuring-repo/blob/new_metrics/measurement_configs/measurement/brain.yaml).
This is the only repository, where the output is stored in a database. Either it is sqlite or MongoDB.

## Off The Pipeline

Here are some repos that aren't part of the pipeline.
Not because they are not yet integrated, but because they aren't meant to be there.

### [Torch Volume Slicing Dataset](https://github.com/DL4XRayTomoImaging-KIT/TVSD)
This is custom dataloader, that just adds some quality of life features to work with large TIFF files.
It works without markup, and for segmentation or classification markup.
It supports random access slicing on different axes (as an option -- all in one dataset, so the length of the dataset is sum of it's dimensions), it supports mem-mapping files, and memory-saving loading of the segmentation markup (where the part which is completely zeros isn't stored).
It also supports re-balancing empty and segmented slices, and automated cropping close to the segmentation.
It is a library which is used under the hood in [the repository for NN training](https://github.com/DL4XRayTomoImaging-KIT/training-repo).

### [Universal File Reader](https://github.com/DL4XRayTomoImaging-KIT/UnivRead3D)
This is again a library which creates universal interfaces to read different useful file formats (`.nii`, `.tif`, `.nrrd`). 
We simply tested and selected the best library to do so, and supported mem-mapping when possible.
Also works with batched 2D files, when they are serially named in one folder.
Used under the hood of [TVSD](https://github.com/DL4XRayTomoImaging-KIT/TVSD)

### [File List Expander](https://github.com/DL4XRayTomoImaging-KIT/FileListExpander)
Well, I just got tired of different ways the files are stored and loaded. 
Sometimes different names in one folder, sometimes same names in different folders, sometimes even different second-level folder names, then same folders, than same filenames again.
Ah, and you also need to exclude from processing every file, which hass `C2H5OH` in it's name because poor froggo was alcoholic.
So, it is a library, which takes folder name, recursively goes through it to find files that match or exclude some expressions.
But it also can take in a list of files, a text file with list of files, or a wildcarded expression.
Also, this can generate new filenames for the result of batch processing: adding prefix, changing extension, moving to another folder.
Checking if files already exist and lazy calculations included (will not overwrite results and provide you list of files not to process).
Ah, and also it already have argparser and can be easily appended as a parsing group to your parser. ~~micdrop~~

A short overview of basic options is in [this jupyter notebook](https://github.com/DL4XRayTomoImaging-KIT/FileListExpander/blob/main/demo.ipynb), but this lib is used almost everywhere in the pipeline, and is quite readable IMO.

### [3D Data Reviewer](https://github.com/DL4XRayTomoImaging-KIT/Review3D)
This repo simply generates requested view for each volume provided (batched input available, as usual).
This can be either three middle planes (good to briefly overview the whole specimen and positioning), bounding box (good for checking if something pokes out of the volume, e.g., after cropping), or series of slices along the first axis (good for quality and reconstruction review).
Each view is generated as a single png file with multiple panes included.
