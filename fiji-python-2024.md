# Virtual I2K 2024: Fiji + Python

The Python software ecosystem has become a platform of choice for many image analysts, providing the Python scientific stack, deep learning toolkits, and visual tools like napari. In this workshop, you will learn how to combine Fiji with Python-based tools, including the napari user interface, as well as Python scripts, REPLs, and Jupyter notebooks. You will learn how to run Fiji/ImageJ commands such as TrackMate on input image data from napari, sending the computation results (such as images, regions of interest, and tracks) back to napari for further visualization and analysis.

**This workshop assumes users**:
* Have existing Fiji knowledge - at a minimum, you should know what you want to *do* with Fiji.
* Have access to `mamba` (Installation instructions [here](https://mamba.readthedocs.io/en/latest/)) on a terminal on your machine.
* Have *some interest* in using the Python programming language.

![](https://media.imagej.net/napari-imagej/0.1.0/front_page.png)

*`napari` and ImageJ, working side by side*

## Step 1: Install the necessary components

This single line will install all components necessary for this workshop, including:

* Python 3.11
* [`pyimagej`](https://py.imagej.net/), the integration layer between Fiji/ImageJ and Python
* Java 11, which is necessary to run Fiji/ImageJ
* [`napari`](https://napari.org), a popular data viewer for Python 
* [`napari-imagej`](https://napari.imagej.net), the integration layer between Fiji/ImageJ and `napari`

`mamba create -n fiji-python -c conda-forge python=3.11 napari-imagej=0.1.0 openjdk=11`

This commmand will take a few minutes as `mamba` downloads and installs the components. Once it completes, you can *activate* the environment with:

```bash
 $ mamba activate fiji-python
```

## Step 2: Create a Fiji instance!

To ensure your environment is properly set up, let's create a Python file `hello_fiji.py`. This file will be used to create an ImageJ2/Fiji instance, and to print the version 

```python
import imagej

ij = imagej.init()
print(f"Successfully initialized ImageJ {ij.getVersion()}")
```

Running the file, you should see something like the following (a different version is possible) printed out. Note that this can take up to a few minutes, depending on your internet connection, as PyImageJ downloads the latest ImageJ2:

```bash
> python hello_fiji.py
Successfully initialized ImageJ 2.16.0/1.54g
```

We now have an ImageJ2 instance, however it does not contain any of the plugins that come with Fiji. To obtain all of those as well, we can add a parameter to the call `imagej.init()` to tell PyImageJ to include Fiji as well. Note, again, that this can take up a few more minutes, as PyImageJ downloads another ImageJ version:

```python
import imagej

ij = imagej.init("sc.fiji:fiji:2.15.0")
print(f"Successfully initialized Fiji {ij.getVersion()}")
```
Note the new parameter `"sc.fiji:fiji:2.15.0"` - it tells PyImageJ to install Fiji v2.15.0, which will get us a bunch of useful ImageJ/Fiji plugins in addition to ImageJ2/ImageJ. We'll use this Fiji installation for the rest of the workshop!

```bash
> python hello_fiji.py
Successfully initialized Fiji 2.15.0/1.54f
```

### _Tip: Interactive mode_

If you want to experiment with your Fiji installation before moving on, try running `python` with the `-i` flag, which will provide you with an interactive [REPL](https://en.wikipedia.org/wiki/Read%E2%80%93eval%E2%80%93print_loop) after your script finishes running. You can type `exit()` to quit! 

```bash
$ python -i hello_fiji.py
```

## Step 3: Data Transfer

Nearly all scientific applications in Python are built around NumPy's `ndarray`s, or structures that behave like them.

Unfortunately, ImageJ/Fiji has never heard of an `ndarray`, and instead operates on its own image structures.

Fortunately, PyImageJ provides robust API to transfer data between ImageJ/Fiji structures and `ndarray` structures. The subsections below describe both directions.

### Step 3.1: From Fiji to Python

If you have an image `img` from Fiji that you'd like to display in Python, you can convert it into an `ndarray` using the syntax `ij.py.from_java(img)`.

In the script below, an example image `j_img` from Fiji is created using the `IOService`, accessed through `ij.io()`. We then call `ij.py.from_java` to convert it, and use `napari` to display it.

```python
import imagej
import napari

ij = imagej.init("sc.fiji:fiji:2.15.0")
# If you have an image from Fiji...
j_img = ij.io().open("https://media.imagej.net/workshops/data/3d/hela_nucleus.tif")

# ...you can convert it to a Python ndarray...
p_img = ij.py.from_java(j_img)
# ...where you can do more cool things!
viewer, layer = napari.imshow(p_img)

napari.run()
```

*Note: There is much more fun to be had with napari in the sections below!*

### Step 3.2: From Python to Fiji

Just as `ij.py.from_java(img)` transfers data from ImageJ/Fiji data structures to NumPy `ndarray`s, `ij.py.to_java(img)` transfers data in the **other direction**. As the goal of PyImageJ is to execute Fiji functionality *on data in Python*, you'll need this method often.

In the script below, an example image `p_img` is generated from scikit-image. We then call `ij.py.to_java` to convert it, and display it in Fiji.

```python
import imagej
import napari
from skimage.io import imread

ij = imagej.init("sc.fiji:fiji:1.15.0")

# If you have a ndarray in Python...
p_img = imread("https://media.imagej.net/workshops/data/3d/hela_nucleus.tif")

# ...you can convert it to Java...
j_img = ij.py.to_java(p_img)
# ...to do things in Fiji...
j_gaussed = ij.op().filter().gauss(j_img, 10)

# ...and then bring it back to Python for display!
gaussed = ij.py.from_java(j_gaussed)
viewer, layer = napari.imshow(gaussed, title="gaussian blur")
# Uncomment the line below to compare the before/after in napari
# viewer.add_image(p_img, name="original image")

napari.run()
```

### A note on `ij.py`

You'll likely find that the functions under `ij.py` work on *many* different types of data. Often times, when you're working with PyImageJ you'll see errors like:
```
No matching overloads found for net.imagej.ops.filter.FilterNamespace.gauss(numpy.ndarray,int), options are:
	public net.imglib2.RandomAccessibleInterval net.imagej.ops.filter.FilterNamespace.gauss(net.imglib2.RandomAccessibleInterval,net.imglib2.RandomAccessibleInterval,double)
	public net.imglib2.RandomAccessibleInterval net.imagej.ops.filter.FilterNamespace.gauss(net.imglib2.RandomAccessibleInterval,net.imglib2.RandomAccessibleInterval,double[])
	public net.imglib2.RandomAccessibleInterval net.imagej.ops.filter.FilterNamespace.gauss(net.imglib2.RandomAccessibleInterval,net.imglib2.RandomAccessibleInterval,double,net.imglib2.outofbounds.OutOfBoundsFactory)
	public net.imglib2.RandomAccessibleInterval net.imagej.ops.filter.FilterNamespace.gauss(net.imglib2.RandomAccessibleInterval,double)
	public net.imglib2.RandomAccessibleInterval net.imagej.ops.filter.FilterNamespace.gauss(net.imglib2.RandomAccessibleInterval,net.imglib2.RandomAccessible,double[])
	public net.imglib2.RandomAccessibleInterval net.imagej.ops.filter.FilterNamespace.gauss(net.imglib2.RandomAccessibleInterval,net.imglib2.RandomAccessibleInterval,double[],net.imglib2.outofbounds.OutOfBoundsFactory)
	public net.imglib2.RandomAccessibleInterval net.imagej.ops.filter.FilterNamespace.gauss(net.imglib2.RandomAccessibleInterval,double[])
```

In this example traceback, we can see from the first line that we tried to pass an `ndarray` to a method from ImageJ. Always make sure that `ij.py` methods are being used before and after calling ImageJ functionality!

### _Tip: Use xarray to preserve your metadata_

If know that you **always** want to have metadata on your images you can use the `ij.py.to_xarray()` method to specify an `xarray.DataArray` output. This method works on both Java images and NumPy arrays, with limited support for dimension reordering.

## Step 4: SciJava Scripts

ImageJ/Fiji enables **reproducibility** and **sharing** through _SciJava scripts_. If you've used Fiji before, you've likely written some yourself! (If you want some reading material for later, check out [this guide](https://syn.mrc-lmb.cam.ac.uk/acardona/fiji-tutorial/#open-script-editor) by Albert Cardona). `ij.py` once again provides a mechanism, `ij.py.run_script`, to run any existing SciJava script.

Here, we'll utilize an existing script for [Richardson-Lucy deconvolution](https://en.wikipedia.org/wiki/Richardson%E2%80%93Lucy_deconvolution), an algorithm that ImageJ Ops performs quite well. Take the code written below, and place it within a file `decon.groovy`:

```groovy
#@ OpService ops
#@ ImgPlus img
#@ Integer iterations(label="Iterations", value=15)
#@ Float numericalAperture(label="Numerical Aperture", style="format:0.00", min=0.00, value=1.45)
#@ Integer wavelength(label="Emission Wavelength (nm)", value=457)
#@ Float riImmersion(label="Refractive Index (immersion)", style="format:0.00", min=0.00, value=1.5)
#@ Float riSample(label="Refractive Index (sample)", style="format:0.00", min=0.00, value=1.4)
#@ Float lateral_res(label="Lateral resolution (μm/pixel)", style="format:0.0000", min=0.0000, value=0.065)
#@ Float axial_res(label="Axial resolution (μm/pixel)", style="format:0.0000", min=0.0000, value=0.1)
#@ Float pZ(label="Particle/sample Position (μm)", style="format:0.0000", min=0.0000, value=0)
#@ Float regularizationFactor(label="Regularization factor", style="format:0.00000", min=0.00000, value=0.002)
#@output ImgPlus result

import net.imglib2.FinalDimensions
import net.imglib2.type.numeric.real.FloatType

// convert input parameters into meters
wavelength = wavelength * 1E-9
lateral_res = lateral_res * 1E-6
axial_res = axial_res * 1E-6
pZ = pZ * 1E-6

// create synthetic PSF
psf_dims = new FinalDimensions(img)
psf = ops.create().kernelDiffraction(
	psf_dims,
	numericalAperture,
	wavelength,
	riSample,
	riImmersion,
	lateral_res,
	axial_res,
	pZ,
	new FloatType()
)

// convert input image to 32-bit and deconvolve with RTLV
img_f = ops.convert().float32(img)
result = ops.deconvolve().richardsonLucyTV(img_f, psf, iterations, regularizationFactor)
```

Using `ij.py.run_script`, all we have to do is read in the script, and provide our arguments:

```python
import imagej
import napari
from skimage.io import imread

ij = imagej.init("sc.fiji:fiji:2.15.0")
# If you have an image in Python...
input = imread("https://media.imagej.net/workshops/data/3d/hela_nucleus.tif")

# ...and a SciJava script...
with open("./decon.groovy", "r") as f:
    script = "".join(f.readlines())

# ...you can assign that image to the script parameter...

args = {
    "img": input
}
# ...and pass it directly to the script!
result_map = ij.py.run_script("groovy", script, args)

# Note that the result is still a Java object, so we have to convert it back to Python
result = ij.py.from_java(result_map.getOutput("result"))
viewer, layer = napari.imshow(result, title="deconvolved")
# Uncomment the line below to compare the before/after in napari
# viewer.add_image(input, name="original")

napari.run()
```

Using scripts like this maximizes portability - you can write a workflow in a single SciJava script, and it can be called from Python, and within Fiji!

## Step 5: napari-imagej

We've used napari to display the outputs of our initial explorations into PyImageJ, but ImageJ/Fiji are also graphically accessible through napari using the `napari-imagej` plugin. This napari plugin was designed to remove much of the hassle involved in executing ImageJ/Fiji functionality from Python - it handles all of the data conversion, providing the appearance of pure ImageJ/Fiji integration.

To use `napari-imagej`, launch napari by typing `napari` on your terminal. Once napari opens, go to the `Plugins` dropdown menu and click on the `ImageJ2 (napari-imagej)` menu item. `napari-imagej` will initialize the ImageJ2 instance for you, becoming "enabled" once the buttons become enabled.

![](https://media.imagej.net/napari-imagej/0.1.0/startup.gif)

Using `napari-imagej`, we have access to all* of ImageJ/Fiji, much of which now available through a seamless napari interface. As a first look at this interface, let's run the exact same gaussian blur that we ran in Step 2, using `napari-imagej`:

1. Download the image that we've been using so far: https://media.imagej.net/workshops/data/3d/hela_nucleus.tif
2. From your computer's `Downloads` folder, drag and drop the image onto the napari viewer pane.
3. In napari-imagej's search bar, type `gauss`.
4. Under the `Ops` dropdown, you'll find `filter.gauss(img "out"?, img "in", number "sigma", outOfBoundsFactory "outOfBounds"?) -> (img "out"?)`. Double click this entry to bring up the parameter selection dialog.
5. For the input, select `hela_nucleus`, and for the `sigma`, enter `10`. Just as we specified these fields as optional when scripting with PyImageJ, we can leave them blank here.
6. Click `Ok`, and wait for the computation to finish. Note that in the `activity` pane in the bottom right hand corner of the napari window, `filter.gauss` will appear to notify you of its status.

Using this mechanism, you can run any ImageJ2 Command or Op, as well as any SciJava Script (as seen in the next step), using a pure napari interface, and none of the extra `ij.py` function calls to make everything work out!

**When we say **all** of Fiji, this is true for Windows and Linux - unfortunately, MacOS cannot run the UI of ImageJ/Fiji while concurrently allowing interactive access to Python or napari. This also means that MacOS users will find themselves unable to run the final section of this workshop (although we've tried to make the rest of this workshop accessible by avoiding the ImageJ/Fiji UI). This has been described in issues like [this one](https://github.com/imagej/pyimagej/issues/298). Many software engineers have lost many hours trying to combat this issue, but rest assured, we are still unfazed in our goal to fix it!*

## Step 6: Running our script again

SciJava Scripts can be run in the `napari-imagej` UI, in addition to the utility they offer in ImageJ/Fiji and in Python scripts. With a bit of organization, described in the steps below, we can show `napari-imagej` how to find these scripts:
1. Make a new `scripts` folder in your current directory. This directory is what ImageJ/Fiji looks for to discover all user scripts, and we will tell `napari-imagej` to look for this folder. Take the `decon.groovy` script that we wrote in Step 4, and place it within that `scripts` folder.
2. Access the settings window within `napari-imagej`, by clicking the "gear" icon on the right. ImageJ/Fiji needs to know which directory **contains** the `scripts` folder, (i.e. **not** the `scripts` folder itself), and so the current directory, containing `hello_fiji.py`, is the one that we need to specify. Set the "ImageJ Base Directory" setting to the directory containing the `scripts` subfolder and `hello_fiji.py`.
3. Restart napari and napari-imagej. On restart, ImageJ/Fiji will use this setting from `napari-imagej` to find our script contained within.

Now that ImageJ/Fiji knows of our `decon.groovy` script, it will become available within the `napari-imagej` search results under the `Commands` dropdown. You will find the script by typing `decon` within the `napari-imagej` search bar. Click this result twice to open the parameter selection window. Note that `napari-imagej` is able to read the script parameters and creates either a pop window or an integrated widget.

Select `hela_nucleus` in the parameter selection window, and then click `Ok` to run `decon.groovy`. While it runs, you can click the "activity" pane in napari to show current progress. Once it finishes, you will see the deconvolved result back in napari!

## Step 7: Tracking with TrackMate in napari-imagej

In the last step of this workshop, we will show how `napari-imagej` enables users to create complex workflows, using the UIs of and toolkits from both Fiji and napari. **This Step is currently inaccessible to MacOS users; please see Addendum 1 for more information**.

To access our Fiji functionality, we must then alter the ImageJ2 installation used by `napari-imagej`, by clicking on the settings button:

![](https://media.imagej.net/napari-imagej/0.1.0/settings_wheel.png)

Clicking this button will display a dialog where, among other things, we can enter a Fiji version, just like we did with PyImageJ.

**Under the "ImageJ directory or endpoint" setting, enter `sc.fiji:fiji:2.15.0`.**

A restart of napari, by closing the window and then restarting napari and `napari-imagej`, will provide us with Fiji.

To run TrackMate itself within Fiji, follow the tutorial [here](https://napari.imagej.net/en/0.1.0/examples/trackmate.html#preparing-the-data), where the process is outlined more thoroughly than we could here.

## Fin

This completes the Fiji + Python workshop! Using the ideas presented in this workshop, you now know how to:

* Launch and configure ImageJ/Fiji within a Python script, and in napari - note that these mechanisms work equally well in a Python REPL, or in a Jupyter Notebook
* Transfer data between Python and Java equivalents, enabling the execution of ImageJ/Fiji routines on data stored in Python objects, and vice versa
* How to run scripts written for ImageJ/Fiji seamlessly within a Python script, or in napari, enabling workflow distribution, portability, and reproducibility.

We hope that, using the above concepts, you can now extend the presented scripts to integrate new, exciting Python tools with your existing Fiji needs!

For more information and help, check out:
* https://py.imagej.net for more information about PyImageJ
* https://napari.imagej.net for more information about `napari-imagej`
* https://forum.image.sc for interactive help about general image analysis problems - we're happy to help you resolve issues!

## Addendum 1: Launching the ImageJ/Fiji UI

One of the more useful features of PyImageJ is the ability to launch and interact with the ImageJ/Fiji user interface. For accessibility, this workshop uses only "headless" (i.e. not requiring the UI) functionality with the exception of Step 7. We designed the workshop as such to maximize accessibility, as [inherent limitations](https://github.com/imagej/pyimagej/issues/298) in MacOS limit the potential of UI display. Nevertheless, UI access is still quite powerful, and we dicuss the three flavors of UI access here.

PyImageJ offers two different mechanisms for launching a Fiji UI; users who wish to use **either** of these mechanisms must indicate so within the call to `imagej.init`:
* `image.init(mode="gui")` tells PyImageJ to launch the ImageJ/Fiji UI **and block the Python thread until the user closes it**. If a user has PyImageJ installed, this is often the fastest way to launch a particular version of ImageJ/Fiji for quick experimentation.
```python
import imagej
ij = imagej.init("sc.fiji:fiji:2.15.0", mode="gui")
```
* `image.init(mode="interactive")` tells PyImageJ to launch the ImageJ/Fiji UI **but returns control of the Python thread back to the user**. *This mechanism is unavailable on MacOS* due to the threading limitations.
```python
import imagej
ij = imagej.init("sc.fiji:fiji:2.15.0", mode="interactive")
```

In addition, as shown in Step 7 of this workflow we can launch both the napari and ImageJ/Fiji UIs using a button within `napari-imagej`, using the button highlighted below. *This mechanism is **also** unavailable on MacOS* due to the threading limitations.
![](https://media.imagej.net/napari-imagej/0.1.0/settings_gui_button.png)

## Addendum 2: Pitfalls in image conversion

While NumPy `ndarray`s provide many benefits throughout the Python ecosystem, they do not have a defined mechanism for image metadata. This means that many images stored within numpy arrays rely on [conventions](https://scikit-image.org/docs/stable/user_guide/numpy_images.html#coordinate-conventions) to convey the meaning of each dimension, but these conventions pose two problems for Python + Fiji integration:

	1. The conventions do not align with the conventions of ImageJ - for example, grayscale images are represented as `(Y, X)` with the NumPy conventions, but as `(X, Y)` within ImageJ/Fiji.
	2. There is no way to enforce these conventions.

To solve the first problem, PyImageJ converts different Python data structures differently, *depending on whether or not they have dimensional metadata*. Some data structures, like [`xarray`](https://docs.xarray.dev/en/stable/)s, have that data, and PyImageJ can use that data to correctly order the dimensions when converting to Java data structures. If the data structure is a pure NumPy `ndarray` though, it does not have this metadata, and PyImageJ permutes the dimensions according to those conventions.

To solve the second problem, we suggest using metadata-rich data structures like `xarray` to encode the dimensions within your datasets.

## Addendum 3: Jupyter Notebook

Jupyter Notebooks provide a powerful interactive setting for constructing sharable workflows, and we can utilize PyImageJ in Jupyter just as we have earlier in the workshop.

To experiment with Jupyter, we must first install it, which we can do with the following command:

```bash
$ mamba install -y -c conda-forge jupyter
```

We then start Jupyter, which will start a Jupyter server, opened in your default browser:

```bash
$ jupyter notebook
```

Create a new notebook here by selecting `File->New->Notebook`. Select `Python 3 (ipykernel)` as the kernel.

In the first cell, you can initialize an ImageJ/Fiji instance, *exactly as we did before*:

```python
import imagej
ij = imagej.init("sc.fiji:fiji:2.15.0")
```
Jupyter users will find PyImageJ utility function `ij.py.show` particularly useful, as it will display an image in a matplotlib plot. This script, modified from section 3.2, can be pasted directly into the next cell.

```python
from skimage.io import imread

ij = imagej.init("sc.fiji:fiji:1.15.0")

p_img = imread("https://media.imagej.net/workshops/data/3d/hela_nucleus.tif")
# Convert our image to Java and do stuff in Fiji
j_img = ij.py.to_java(p_img)
j_gaussed = ij.op().filter().gauss(j_img, 10)

# Then convert our image back to Python for display
gaussed = ij.py.from_java(j_gaussed)
ij.py.show(gaussed[30, :, :])
```

Note that `ij.py.show` is capable of showing images stored in *both* Python *and* Java. Can you edit the cell to have PyImageJ display the same slice without the conversion back into Python?

## Addendum 4: `Mesh` visualization in napari

`napari-imagej` can additionally convert Fiji surface structures into napari `Surface` layers, providing a convenient solution for mesh/surface visualization using the napari viewer. The SciJava script written below, which converts a dataset containing a single structure in the foreground into a surface, showcases this conversion. 

```python
#@ OpService ops
#@ Img img
#@ Float (label = "Isolevel", style = "format:0.00", min = 1.0, value = 1.0) isolevel
#@output net.imagej.mesh.Mesh output

from net.imglib2.type.logic import BitType

def apply_isolevel(image, isolevel):
    """Apply the desired isolevel on the input image.

    Apply a desired isolevel (i.e. isosurface) value on the input image,
    returning a BitType image that can be used with the marching cubes Op.

    :param image:

        Input ImgPlus.

    :param isolevel:

        Input isolevel value (float).

    :return:

        An ImgLib2 Mesh at the specified isolevel.
    """
    if isolevel > 1.0:
        isolevel -= 1
    val = image.firstElement().copy()
    val.setReal(isolevel)
    bin_img = ops.create().img(image, BitType())
    ops.threshold().apply(bin_img, image, val)

    return ops.geom().marchingCubes(bin_img)


output = apply_isolevel(img, isolevel)
```

We will run this script on [this image]("https://workshops.imagej.net/images/hela_nucleus_8_bit.tif), the same dataset used throughout the workshop, but reduced to unsigned 8-bit integers to avoid [this issue](https://github.com/imagej/napari-imagej/issues/276) which affects our version of `napari-imagej`. Use the following steps to set up this script for execution on the sample dataset:

1. Create a new file `mesh.py` in the `scripts` folder where the other scripts were placed, and copy the above script into the new file. 
2. Start napari and `napari-imagej` if they are not running
3. Load our reduced image into the viewer
4. Determine the surface `isolevel`, **the value below which a particular location will be considered "below" the mesh**. You can mouse over the image using napari to find an appropriate isolevel near the edge of the main structure.
5. Search for `mesh` in the `napari-imagej` search bar. The script will appear under the `Commands` result tab.
6. Double-click the `mesh` search result. Specify:
  *  `hela_nucleus_8_bit` as the `img` parameter
  * Your determined vlaue as the `isolevel` parameter.
  
Once the script finishes, the resulting `net.imagej.mesh.Mesh` will be converted into a napari `Surface` layer and will be displayed in the viewer. You can toggle the 3D rendering capabilities of napari-imagej using a button described [here](https://napari.org/stable/howtos/layers/surface.html#d-rendering).
