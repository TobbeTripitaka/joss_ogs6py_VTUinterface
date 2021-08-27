---
title: 'ogs6py and VTUinterface: streamlining OpenGeoSys workflows in Python'
tags:
  - Python
  - physics
  - THMC
  - VTU
  - time-series
  - sensitivity analysis
  - uncertainty quantification
  - OpenGeoSys

authors:
  - name: Jörg Buchwald^[corresponding author]
    orcid: 0000-0001-5174-3603
    affiliation: "1, 2" # (Multiple affiliations must be quoted)
  - name: Olaf Kolditz
    orcid: 0000-0002-8098-4905
    affiliation: "1, 3, 4"
  - name: Thomas Nagel
    orcid: 0000-0001-8459-4616
    affiliation: "2, 4"
affiliations:
 - name: Helmholtz Center for Environmental Research - UFZ, Leipzig, Germany
   index: 1
 - name: Technische Universität Bergakademie Freiberg, Germany
   index: 2
 - name: Technische Universität Dresden, Germany
   index: 3
 - name: TUBAF-UFZ Center for Environmental Geosciences, Germany
   index: 4
date: 12 May 2021
bibliography: paper.bib

---

# Summary

[ogs6py](https://github.com/joergbuchwald/ogs6py) is a Python interface for the open-source package OpenGeoSys [@Bilke2019], a finite element code
for the simulation of multi-field processes in fractured porous media.
In conjunction with [VTUinterface](https://github.com/joergbuchwald/VTUinterface) it is possible to streamline modeling workflows
in Jupyter Notebooks using Python.
With this article, we wish to introduce two new Python modules that facilitate
the pre- and post-processing of finite element calculations of OpenGeoSys and thus
make this code more accessible to the community. Their use is demonstrated along workflows typically
encountered by the modeler, including the variation of parameters, boundary conditions,
or solver settings, the verification of simulation results by comparison to analytical
solutions, the set-up and evaluation of ensemble runs, the rapid analysis of results by line plots,
time series, or transient contour plots.

# Statement of need

Driven by its ease-of-use and flexibility as an open-source 
dynamic language, its vast modular ecosystem, the development of powerful plotting
libraries and the Jupyter Notebook technology, Python became a widely used common framework for 
scientific data analysis in the modeling community during the past decade.
However, the attractiveness of Python is not just limited to post-processing. 
E.g., with the Python wrapper for GMSH [@geuzaine2009gmsh] or the tool meshio [@nico_schlomer_2021_4745399] also pre-processing tasks can
be easily conducted without leaving the IPython command prompt. The usability of any modeling package
is therefore greatly enhanced if Python bindings are provided. In fact, 
while many open-source tools effectively forced the user to learn a singular syntax
for interacting with the software, Python bindings allow control over such tools from 
within the Python world and thus open them up for a wider community of users.

Here, we are particularly addressing the open-source code OpenGeoSys (OGS) [@Bilke2019] version 6. It is our aim
to facilitate both pre- and post-processing workflows utilizing the Python ecosystem. 
This aim was not the least inspired by the desire to facilitate setting up, controlling and
evaluating ensemble runs [@Buchwald2020;@Chaudhry2021] but has now taken on a wider perspective of general 
software usability. There exists already a similar Python interface "ogs5py" for OGS version 5 [@muller2021ogs5py]. 
However, the differences in many concepts, like the use of XML input files, required an entirely new package to be built from scratch.

As standard output format, OpenGeoSys uses VTK unstructured grid files (VTU) as time slices stacked together by a PVD file.
These can be analyzed typically using Paraview [@ahrens2005paraview]. For interactive use the Python 
wrapper for VTK [@schroeder2000visualizing] and some other tools like PyVista [@sullivan2019pyvista] or Mayavi [@ramachandran2011mayavi] 
are available facilitating an easier access to the VTK library.
While the direct use of the VTK library is quite cumbersome for basic tasks, like reading data for a given point set, especially when interpolation between grid points is also required, the latter packages focus mainly on 3D visualization. However, the _bread and butter_ business of a finite-element-modeler often consists of the extraction of single- or multiple point time-series data.
To our knowledge the named packages (with the exception of Paraview) don't have file support for PVDs or time series data, yet [@pvdissue; @timeseriesissue].

# Features

ogs6py allows creating complete OGS configuration files from scratch, altering existing files, running simulations and parsing OGS log files.
The following example demonstrates some basic functionalities.
The complete example demonstrating a common ogs6py/VTUinterface workflow on a coupled thermo-hydro-mechanical (THM) problem of a tunnel excavation followed by the
emplacement of a heat-emitting canister can be found in a 
[Jupyter notebook](https://github.com/joergbuchwald/joss_ogs6py_VTUinterface/blob/master/demo/paper_ogs6py_vtuio.ipynb) located in the project repository.


To read in an existing project file and to specify an output name an instance of OGS needs to be created.

```python
model = OGS(INPUT_FILE="tunnel_ogs6py.prj", PROJECT_FILE="tunnel_exc.prj")
```

A project file can then be altered by commands for adding blocks, removing or replacing parameters like

```python
model.replace_phase_property(mediumid=0, phase="Solid",
        name="thermal_expansivity", value=a_s)
```

or


```python
model.replace_text("tunnel_exc", xpath="./time_loop/output/prefix")
```

The project file can be written to disk with

```python
model.write_input()
```

and OGS can be executed by calling the `run_model()` method:

```python
model.run_model(path="~/github/ogs/build_mkl/bin",
        logfile="excavation.log")
```

OGS produces PVD and VTU files that can be handled with VTUinterface:

```python
pvdfile = vtuIO.PVDIO("tunnel_exc.pvd", dim=2)
```

One of the most significant features of VTUinterface is the ability to deal with PVD files as time series data.
E.g., the following command reads in the VTU point field "pressure" at point "pt0", defined in a dictionary,
using nearest neighbor interpolation.

```python
excavation_curve = pvdfile.read_time_series("pressure",
        interpolation_method="nearest",  pts={"pt0": (0.0,0.0,0)})
```

The result can directly be plotted using matplotlib (\autoref{fig:1}).
The time axis can be retrieved from the PVD file as well.

```python
plt.plot(pvdfile.timesteps, excavation_curve["pt0"] / 1e6)
plt.xlabel("$t$ / d")
plt.ylabel("$p$ / MPa");
```

![Plots demonstrating the usage of VTUinterface: Deconfinement curve extracted as timeseries from a PVD file of excavation simulation (left). Contour plot of pressure distribution generated with VTUinterface and matplotlibs `tricontourf()` showing the thermal pressurization during the heating phase (right).\label{fig:1}](fig1.png){ width=100% }


![Spatial pressure distribution generated with VTUinterface from a linear point set array using three different grid interpolation methods (left). Relative convergence plot showing the numerical behaviour over ten time steps extracted using the log file parser of ogs6py (right).](fig2.png){ width=100% }


This brief overview shows only some of the functionalities coming with ogs6py and VTUinterface.
Further developments will focus on extending functionalities with a focus on built-in checks to ensure that only valid input files are generated.


# Applications

Both introduced packages are with 1-2 years of age relatively new. However, the adoption process in the OpenGeoSys community is gearing up.
E.g., a [YouTube video](https://www.youtube.com/watch?v=eihNKjK-I-s) was published explaining their use, both tools are also used for teaching 
at the TU Bergakademie Freiberg and they were also extensively utilized in two recent peer-reviewed publications [@buchwald2021improved; @Buchwald2020].

# Acknowledgements

We acknowledge contributions from Tom Fischer, Dmitry Yu. Naumov, Dominik Kern and Sebastian Müller during the genesis of this project.
The funding through the iCROSS-Project (Integrity of nuclear waste repository systems – Cross-scale system understanding and analysis) 
by the Federal Ministry of Research and Education (BMBF, grant number 02NUK053E) and Helmholtz Association (Helmholtz-Gemeinschaft e.V.) 
through the Impulse and Networking Funds (grant number SO-093) is greatly acknowledged. This work was in part funded by the Deutsche 
Forschungsgemeinschaft (DFG, German Research Foundation) under grant number NA1528/2-1.

# References
