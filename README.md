# Forest Structural Complexity Tool
### Created by Sean Krisanski
## Purpose of this tool

This tool was written for the purpose of allowing plot scale measurements to be extracted automatically from most
high-resolution forest point clouds from a variety of sensor sources. Such sensor types it works on include
Terrestrial Laser Scanning (TLS), Mobile Laser Scanning (MLS), Terrestrial Photogrammetry, Above and below-canopy
UAS Photogrammetry or similar. Very high resolution Aerial Laser Scanning (ALS) is typically on the borderline of what
the segmentation tool is capable of handling at this time. If a dataset is too low resolution, the segmentation model
will likely label the stems as vegetation points instead.

There are also some instances where the segmentation model has not seen appropriate training data for the point cloud.
This will be improved in future versions, as it should be easily fixed with additional training data.



## Citation
### If you find this tool helpful or use this tool in your research, please cite:

**The semantic segmentation tool is described here:**
\
Krisanski, S.; Taskhiri, M.S.; Gonzalez Aracil, S.; Herries, D.; Turner, P. Sensor Agnostic Semantic Segmentation of
Structurally Diverse and Complex Forest Point Clouds Using Deep Learning. Remote Sens. 2021, 13, 1413.
https://doi.org/10.3390/rs13081413

\
**The measurement tool is described here:**
\
Krisanski, S.; Taskhiri, M.S.; Gonzalez Aracil, S.; Herries, D.; Montgomery, J.; Turner, P. Forest Structural Complexity Tool - An Open
Source, Fully-Automated Tool for Measuring Forest Point Clouds. Remote Sens. 2021, XX, XXXX. 
https://doi.org/XX.XXXX/rsXXXXXXXX

## Acknowledgements
This research was funded by the Australian Research Council - Training Centre for Forest Value 
(University of Tasmania, Australia).

Thanks to my supervisory team Assoc. Prof Paul Turner and Dr. Mohammad Sadegh Taskhiri from the eLogistics Research
Group and Dr. James Montgomery from the University of Tasmania.

Thanks to Susana Gonzalez Aracil and David Herries from Interpine Group Ltd. (New Zealand), who provided a number of the raw point
clouds and plot measurements used during the development and validation of this tool.


## Contributing
Interested in contributing to this code? Get in touch! This code is likely far from optimal, so if you find errors or 
have ideas/suggestions on improvements, they would be very welcome!

## How to use
It is strongly recommended to have a CUDA compatible GPU (Nvidia) for running this tool. This can be run on CPU
only, but expect inference to take a long time.

### Recommended PC Specifications
**Warning: FSCT is computationally expensive in its current form.** Fortunately, it is still considerably faster than a human 
at what it does.

It should be able to be run on most modern gaming desktop PCs (or particularly powerful laptops), however, it will
take a while if you are running it on a lesser setup than below.

I use the following setup and the computational times are bearable:
- CPU: Intel i9-10900K Overclocked to 4.99GHz all cores.
- GPU: Nvidia Titan RTX (24 GB vRAM)
- RAM: 128 GB DDR4 at 3200 MHz (NOTE: this is often not enough on large point clouds, so I have a 500 GB page file to assist if I run out of RAM).
- SSD: M.2 NVMe 2 TB, 3500 MB/s read,  3000 MB/s write.

Hopefully in time, I'll be able to make this more efficient and less resource hungry.

### User Parameters
#### batch_size
The number of samples in a batch used for the deep learning inference. This number depends on the amount of GPU RAM you
have. If you set this too high, you will run out of GPU RAM. As a rough guide, I can fit 18-20 on an Nvidia Titan RTX GPU with 24 GB GPU
RAM.

#### num_procs
The number of CPU cores you have/wish to use.

ground_veg_cutoff_height=3,  # Any vegetation points below this height are considered to be understory and are not assigned to individual trees.
veg_sorting_range=5,  # Vegetation points can be, at most, this far away from a cylinder horizontally to be matched to a particular tree.
sort_stems=1,  # If you don't need the sorted stem points, turning this off speeds things up. Veg sorting is required for tree height measurement, but stem sorting isn't necessary for general use.
stem_sorting_range=1,  # Stem points can be, at most, this far away from a cylinder in 3D to be matched to a particular tree.
Site='',  # Enter the site name if you wish. Only used for report generation.
PlotID='',  # Enter the plot name/ID if you wish. Only used for report generation.
plot_centre=None,  # [X, Y] Coordinates of the plot centre (metres). If "None" or 0, plot_centre is set as the centre XY coords of the point cloud.
plot_radius=20,  # If 0 m, the plot is not cropped. Otherwise, the plot is cylindrically cropped from the plot centre with plot_radius + plot_radius_buffer.
plot_radius_buffer=3,  # See README.md  This is used for "Intelligent Plot Cropping Mode".
UTM_zone_number=50,  # Set this or the Lat Lon outputs will be incorrect.
UTM_zone_letter='',
UTM_is_north=False,  # If in the northern hemisphere, set this to True.
low_resolution_point_cloud_hack_mode=0,  # See README.md for details. Very ugly hack that can sometimes be useful on low resolution point clouds.
delete_working_directory=True  # Generally leave this on. Deletes the files used for segmentation after segmentation is finished.
                             # You may wish to turn it off if you want to modify the segmentation code so you don't need to run pre-processing every time.



### Intelligent Plot Cropping
The purpose of this mode is to simulate the behaviour of a typical field plot, by not chopping trees in half if they are
at the boundary of the plot radius.

We first trim the point cloud to a radius where the initial trim radius = plot_radius + plot_radius_buffer.
For example, we might want a 20 m plot_radius. If we use a 3 m plot_radius_buffer, the point cloud will be cropped to
23 m radius initially. FSCT will then use the measurement information extracted from the trees in that 23 m radius point
cloud, to check which tree centres are within the 20 m radius. This allows a tree which was just inside the boundary, to
extend 3 m beyond the plot boundary without losing points. If we used a simple radius trim at 20 m, trees which were
just inside the boundary may be cut in half.

This mode is used if plot_radius is non-zero and plot_radius_buffer is non-zero.

### Low resolution hack mode...
This model was trained on relatively high resolution point clouds, so if a stem is of sufficiently low resolution,
it will likely be classified as vegetation instead. Eventually, I will train this model on a larger training dataset
with more examples of sparse point clouds, however, in the meantime, I came up with an ugly hack that
sometimes helps a little when working with low resolution datasets (which this tool was not really designed for).

Low resolution hack mode will copy the input point cloud, jitter the points in random directions by 1 cm, then join this
copied point cloud to the original point cloud. This gives the model more points to work with, which is closer to what
it was trained on. Once inference is complete, the original point cloud is returned.


## References
The deep learning component uses Pytorch https://pytorch.org/ and Pytorch-Geometric 
https://pytorch-geometric.readthedocs.io/en/latest/#

The first step is semantic segmentation of the forest point cloud. This is performed using a modified version of
Pointnet++ https://github.com/charlesq34/pointnet2 using the implementation in Pytorch-Geometric as a starting point
provided here: https://github.com/rusty1s/pytorch_geometric/blob/master/examples/pointnet2_segmentation.py

We make extensive use of NumPy (Harris, C.R., Millman, K.J., van der Walt, S.J. et al. Array programming with NumPy.
Nature 585, 357–362 (2020). https://doi.org/10.1038/s41586-020-2649-2) and Scikit Learn (Scikit-learn: Machine Learning in Python, Pedregosa et al., JMLR 12, pp. 2825-2830, 2011. http://jmlr.csail.mit.edu/papers/v12/pedregosa11a.html)
