# Implementation of PFIO in LIS

## Introduction

The [Land Information System](https://lis.gsfc.nasa.gov/) (LIS) is a software framework for high performance terrestrial hydrology modeling and data assimilation developed with the goal of integrating satellite and ground-based observational data products and advanced modeling techniques to produce optimal fields of land surface states and fluxes.
The framework consists of three modeling components, one of which is the Land Information System (LIS): the modeling system that encapsulates physical models, data assimilation algorithms, optimization and uncertainty estimation algorithms, and high performance computing support. 

The section of the LIS code involved in calculations is embarrassedly parallel and can scale up to thousands of processors. 
The model produces a HISTORY collection. As the number of processor increases, the time to create HISTORY may be several order of magnitude greater than the time for computations. 
That is a major bottleneck for LIS users who want to run the model at higher resolutions and with thousands of processors. 
The main challenge in the output procedure is that the root processor is in charge of gathering the data from all the other processors and writing data into disc (while all the other processors are waiting for the next set of calculations).
It is not possible to scale or speed up this procedure in a distributed environement.

We have [explored several methods](https://github.com/JulesKouatchou/LIS_profiling) to find ways to improve the IO performance in the LIS code.
We finally consider the [GEOS PFIO](https://github.com/GEOS-ESM/MAPL/wiki/PFIO:-a-High-Performance-Client-Server-I-O-Layer) module that is
a parallel I/O tool that was designed to facilitate the production of model netCDF output files (organized in collections) and to efficiently use available resources in a distributed computing environment. 

In this document, we describe how PFIO was implemented in LIS. 
We also show how the new code LIS/PFIO is compiled and run. 
Finally, we present the performance of LIS/PFIO.

## Implementation Steps

### Requirements
The goal was to modify the code and make sure that it can compile and run with or without PFIO.
To achieve it, we made the decision to:
- Add the PFIO module as an external library of LIS. The GEOS MAPL (where PFIO is a subcomponent) was compiled and installed as an external library for LIS.
- Any PFIO related section should be selected a compilation time. We added preprocessing directives throughout the code to facilitate the implementation of PFIO by maintaining the original functionalities of the code.

If the above two requirements are sastified, the same base code can be used by LIS users (in a transparent way),
and the new additions will interfer with any LIS code development.

### New Files

Our main objective was to producde HISTORY (in a netCDF file format) without disrupting any other LIS process.
We used the existing LIS infrastructure and create a new component to receive LIS field arrays 
(as one-dimentional tiles), transform them (as two-dimensional arrays), and pass the transformed arrays to the PFIO I/O Server.
We used as reference the steps contained in the original file `LIS_historyMod.F90` that has all the utility routines for
output procedures. 

One advantage of PFIO is that works with variable names but not with variable ids (as when we directly use netCDF calls).
Our task was to write routines that keep track of variable names (including for dimensions) only.
Also, the PFIO related calls act on local variables only. We did not have to manipulate any global variables.
However, we had to create new local arrays to store all fields while the data are transferred to the PFIO I/O server.

We created the following files to support our implementation of PFIO in LIS:


`core/LIS_PFIO_historyMod.F90`: 
- Contains routines for HISTORY using PFIO calls. The main ones are: 
    - `PFIO_create_file_metadata`: creates the netCDF file metadata and the PFIO History identifier. This sunroutine is called once.
    - `PFIO_write_data`: does basic manipulations of the local data (for instance converting from tile to 2D array) and sends the local data (entire array without slicing) to the PFIO IO Server. This subroutine is called any time the model is ready to write the data.
    - `map_1dtile_to_2darray`: maps the LIS local one-dimensional tiles into a local two-dimensional lat/lon array. This internal mapping routine is needs to be accurate and should ensure that the data are assigned to the appropriate grid points.
- It mimics the steps avalaible in `LIS_historyMod.F90`. 
- The main feauture of this module is the implementation of virtual HISTORY collections to take advantage of the capabilities of PFIO (that works best with multiple file collections). 
- The inclusion of the virtual collections is the most critical feature that makes LIS/PFIO attractive and efficient.

`core/LIS_PFIO_varMod.F90`: 
- Module for defining variables needed to manipulate LIS data with PFIO.

`core/LIS_PFIO_utilsMod.F90`: 
- Module containing PFIO utilitiy routines.

`core/LIS_rcFileReading_mod.F90`: 
- Module containing utility routines to read rc variables and setting default values.

`core/LIS_ftimingMod.F90`: 
- Module containing utility routines to profile any MPI application.

### Modified Files

Files modified mainly to accommodate PFIO:

`arc/Config.pl`: 
- Added a question for selecting PFIO or not.
- If PFIO is selected at compilation, provide the necessary locations to the MAPL include files and libraries.

`offline/lisdrv.F90`:
- Reorganized this file to include two sections separed by preprocessing directives:
    - The code that was in the original file and that works without the use of PFIO.
    - A new code that contains the PFIO statements spawn all the MPI processes, creates the compute nodes and IO nodes (based on the command line parameters) and to drive the model.
- Depending on the choice during the configuration step, one section of the code will be selected for compilation.

`core/LIS_coreMod.F90`: 
- Declare and allocate variables needed by PFIO.
- Do not finalize ESMF if PFIO is used.
- Add preprocessing directives.

`core/LIS_surfaceModelMod.F90`: 
- Include calls from the module `LIS_PFIO_historyMod.F90` for the HISTORY creation.
- Add preprocessing directives to differentiate HISTORY subroutine related to calls involving PFIO or not.

`core/LIS_historyMod.F90`: 

To allow the netCDF data compression parameters to be set at run time in the `lis.config` file, we added the following:
```
    shuffle       = LIS_rc%nc_shuffle        ! NETCDF_shuffle
    deflate       = LIS_rc%nc_deflate        ! NETCDF_deflate
    deflate_level = LIS_rc%nc_deflate_lvl    ! NETCDF_deflate_level
```

`core/LIS_PRIV_rcMod.F90`: 
- Declared new variables:
    - `do_ftiming` (do code profiling?)
    - `n_vcollections` (num PFIO virtual collections)
    - `nc_shuffle`: NETCDF shuffle filter
    - `nc_deflate`: NETCDF deflate filter
    - `nc_deflate_lvl`: NETCDF deflate level

`core/LIS_readConfigMod.F90`: 
- Added calls to read rc file variables for profiling or not, the number virtual collections (only need is PFIO used) and data compression parameters.

`core/LIS_domainMod.F90`:
- Set the variables `LIS_rc%procLayoutx = Pc` and `LIS_rc%procLayouty = Pr` that are needed by the profiling tool.
- Added profiling calls.

Files modified only for profiling:
 
- `core/LIS_metforcingMod.F90`: 
- `core/LIS_domainMod.F90`: 
- `core/LIS_paramsMod.F90`: 

## Compiling the Code

The compiling procedures remain the same as in the original LIS code.
Users need to first run the configuration:

```bash
   ./configure
```

In the dialogue, there is now a new option:

```bash
   Use PFIO to produce nc4 HISTORY (1-yes, 0-no, default=0):
```

where the default value (`0`) will generate a code that works without PFIO, i.e., the original LIS code (with not PFIO library and related calls).

In addition, the folowing configuration settings:

```bash
   NETCDF use shuffle filter? (1-yes, 0-no, default = 1): 
   NETCDF use deflate filter? (1-yes, 0-no, default = 1): 
   NETCDF use deflate level? (1 to 9-yes, 0-no, default = 9): 
```

are no longer needed because the three parameters are now set at run time in the `lis.config` file.

After successfully running the `configure` script, a summary of the settings used will be printed on the screen:  

```
----------------------------------------------------
------------- SUMMARY of the SETTINGS --------------
----------------------------------------------------
                                     Parallelism: 1
                 Use PFIO to produce nc4 HISTORY? 1
                              Optimization level: 2
            Assume little/big_endian data format: 2
                             Use GRIBAPI/ECCODES? 2
Enable AFWA-specific grib configuration settings? 0
                                      Use NETCDF? 1
                                  NETCDF version: 4
                                        Use HDF4? 1
                                        Use HDF5? 1
                                      Use HDFEOS? 1
                                     Use MINPACK? 0
                                    Use LIS-CRTM? 0
                                    Use LIS-CHEM? 0
                                  Use LIS-LAPACK? 0
                              Use LIS-MKL-LAPACK? 0
                                       Use PETSc? 0
----------------------------------------------------
```

When the configuration is done, users need to execute the compilation script:

```bash
   ./compile
```

to create the executable file `LIS`.
Note that if PFIO is not selected during the configuration step, the excutable file will be the standard `LIS` file (without any PFIO reference).

## Running the Code

Here, we assume that the gererated executable has PFIO references.
PFIO offers the option to run the code using either the standard `mpirun` command or an IO server configuration (reserved for producing HISTORY only). 
More infoormation can be obtained from the [PFIO description guide](https://github.com/GEOS-ESM/MAPL/wiki/PFIO:-a-High-Performance-Client-Server-I-O-Layer). 

Because the LIS code only has one collection (one HISTORY file), we implemented the capability to create virtual HISTORY collections, 
where the number of collection is chosen at run time in the `lis.config` file.
The file has the settings:

```
   Profiling Tool:                    .TRUE.  # do you want to profile the code (no as default)? 
   num PFIO virtual collections:      4       # number of virtual collections (1 as default)
   netCDF shuffle filter:             0
   netCDF deflate filter:             1
   netCDF deflate level:              1       # deflate level (0 as default)
```

In the SLURM script, it is important to pass the appropriate parameters from the command line:
- The total number of reserved processors for the experiment.
- The number of IO nodes.
- The number of compute processors.
- The number of backend cores per IO nodes. The total number of backend cores should be at least 2.

If we reserve 308 processors (11 nodes with 28 cores per node) and select 2 IO nodes, the SLURM script could look like:

```
   set       nodes_output = 2
   set       npes_backend = 1
   set num_cores_per_node = 28
   set           tot_npes = 308
   $RUN_CMD $tot_npes LIS --npes_model $comp_npes --oserver_type multigroup --nodes_output_server $nodes_output --npes_backend_pernode $npes_backend
```


While setting PFIO command line paramters, we need to assess the followinG;
- Does the process to produce the HISTORY files is signficantly more expensive than the calculations.
- The elapsed time between the full creation (writing into disk) of two consecutive HISTORY files is less than the model integration time. 
    -  If not, the ouput node might be continually oversubcribed.  
    -  By principle in PFIO, the frontend processors (FPs) forward the data to the backend and they get back to the clients (compute processors) without waiting for the actual writing of the data. It is possible for the clients to send new requests to the FPs while the FPs are still sending data to the backend.
    -  We recommend that we use at least 2 IO nodes with only one backend core per IO node.

Choosing the right command line parameters for a specific experiment is a matter of trials and errors. 
Users need to try different configurations until they determine the one that provide the best timing performane.

## Testing the New Code

We used the same base code to create two different executables: 
- `LIS_org`: does not use PFIO and exerises the original version of the code.
- `LIS_pfio`: relies on PFIO to create HISTORY.

In each of the two cases presented here, we ran experiments as the number of compute processors varies.
In the case of `LIS_pfio`


### Test Case 1

- Land surface model: `Noah-MP.4.0.1`
- Output methodology: `2d gridspace`
- Number of ensembles per tile: 1
- 5901x2801 grid points
- one-day integration with output produced every 1.5 hours (17 files with one record each)
- The fields to be written out are:
    - 80 2D fields
    - 4 3D fields (with 4 levels)
- Each file without compression has a size of 6.43 Gb.
 

| Model Configuration | # compute cores  | # IO Nodes | Calculations | HISTORY | 
|       -----         | ----:            |  ----:     | ----:        | ----:   |
| LIS                 | 504              |            |  61.67       | 1271.18 |
|                     | 1008             |            |  30.78       | 1239.48 |
| LIS/PFIO            |  504             |  2         |  62.34       | 3635.42 | 
|                     |                  |  4         |  62.99       | 1039.46 |
|                     | 1008             |  4         |  30.85       | 2535.18 |
|                     |                  |  6         |  30.91       | 1321.33 |
|                     |                  |  8         |  30.90       |  674.13 |

**Table 1:** LIS and LIS/PFIO: timiming statistics as the number of compute processors and the number of IO node vary. In the LIS/PFIO case, we use 1 backend core per IO node and set the number virtual output collections to the number of IO nodes.


### Test Case 2

- Land surface model: `Noah-MP.4.0.1`
- Output methodology: `2d ensemble gridspace`
- Number of ensembles per tile: 25
- 5901x2801 grid points
- one-day integration with output produced every 3 hours (9 files with one record each)
- The fields to be written out are:
    - 2 2D fields
    - 8 3D fields (with the 25 ensembles)
    - 2 4D fields (with the 25 ensembles and 4 
- Each file without compression has a size of 9.09 Gb.
 

| Model Configuration | # compute cores  | # IO Nodes | Calculations | HISTORY | 
|       -----         | ----:            |  ----:     | ----:      | ----:   |
| LIS                 | 252              |            |  89.95     | 2110.11 |
|                     | 504              |            |  45.00     | 2150.35 |
| LIS/PFIO            |  252             |  2         |  92.56     |  483.04 | 
|                     |                  |  4         |  92.10     |   56.61 |

**Table 2:** LIS and LIS/PFIO: timiming statistics as the number of compute processors and the number of IO node vary. In the LIS/PFIO case, we use 1 backend core per IO node and set the number virtual output collections to the number of IO nodes.

In this particicular test case, PFIO outperforms the original version of the code. 
The main reason is that in the original version of the code, the IO module only manipulates 2D fiels. When there is an additional dimension for instance, the code loops over the new dimension, leading to several SENDs (increase communications) to the root processor and WRITEs to the disc by the root processor (increase of the access to the file system). 
With LIS/PFIO, there is no loop over the additional dimensions as the entire data array is sends to the IO server which will write the global array once.


### Observations

- LIS/PFIO produces files bitwise identical to the original version of the code.
- LIS/PFIO requires less computing resources to achieve the same wall-clock time as the original LIS.
- The use of virtual collections significantly improves the IO performance.
- The configuration nature of PFIO allows using to test several options before find the one that will give the best performance.
- As we increase the model resolution and the IO dominates the model calculations, we expect PFIO to perform better.
- Using the deflation level of 1 is most cost effective in both ORG and PFIO.
    - At such level, PFIO tends to generate a smaller file size.
- **It is also important to note that even if we use the `Simple Server` configuration of PFIO (regular `mpirun` command also used in the regular LIS code), LIS/PFIO still performs better than LIS.**


## Pending Issues

- Implement the `1d tilespace` configuration. I do not know how this is important. It can only be done if PFIO manipulates 1D arrays.
- Allow PFIO to handle the restart file too. This will be the ideal but the way the LIS code is structured (each land surface model has its own calls to create its restart file), I do not think it will be practical to do this.

__I spent time to address the above issues and observed that they were related. I concluded that it is not possible to use PFIO for the `1d tilespace` configuration and for the restart file. The reason is that each local 1D tile (in `1d tilespace`) does not contain data for only a specific 2D subdomain but data that extend to multiple subdomains. With PFIO, we assume that each local process has data a specific subdomain.__

<!---
--->


