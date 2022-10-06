# Implementation of PFIO in LIS

## Introduction

The [Land Information System](https://lis.gsfc.nasa.gov/) (LIS) is a software framework for high performance terrestrial hydrology modeling and data assimilation developed with the goal of integrating satellite and ground-based observational data products and advanced modeling techniques to produce optimal fields of land surface states and fluxes.
The framework consists of three modeling components, one of which is the Land Information System (LIS): the modeling system that encapsulates physical models, data assimilation algorithms, optimization and uncertainty estimation algorithms, and high performance computing support. 
The section of the LIS code involved in calculations is embarrassedly parallel and can scale up to thousands of processors. 
The model produces a HISTORY collection. As the number of processor increases, the time to create the output may be several order of magnitude greater than the time for computations. 
That is a major bottleneck for LIS users who want to run the model at higher resolutions and with thousands of processors. 
The main challenge in the output procedure is that the root processor is in charge of gathering the data from all the other processors and writing data into disc.
It is not possible to scale or speed up this procedure in a distributed environement.

We have [explored several methods](https://github.com/JulesKouatchou/LIS_profiling) to find ways to improve the IO performance in the LIS code.
We finally consider the [GEOS PFIO](https://github.com/GEOS-ESM/MAPL/wiki/PFIO:-a-High-Performance-Client-Server-I-O-Layer) module that is
a parallel I/O tool that was designed to facilitate the production of model netCDF output files (organized in collections) and to efficiently use available resources in a distributed computing environment. 

In this document, we describe how PFIO was implemented in LIS. 
We also show how the new code LIS/PFIO is compiled and run. 
Finally, we present the performance of LIS/PFIO.

## Implementation Steps

### Requirement
The goal is to modify the code and make sure that it can compile and run with or without PFIO.
To achieve it, we made the decision to:
- Add the PFIO module as an external library of LIS. The GEOS MAPL (where PFIO is a subcomponent) was compiled and installed as a LIS external library.
- Any PFIO related section should be selected a compilation time. We added preprocessing directives throughout the code to facilitate the implementation of PFIO by maintaining the original functionalities of the code.


### New Files

`core/LIS_PFIO_historyMod.F90`: 
- Contains routines for HISTORY using PFIO calls. The main ones are: 
    - `PFIO_create_file_metadata`: creates the netCDF file metadata and the PFIO History identifier. This sunroutine is called once.
    - `PFIO_write_data`: does basic manipulations of the local data (for instance converting from tile to 2D array) and sends the local data (entire array without slicing) to the PFIO IO Server. This subroutine is called any time the model is ready to write the data.
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
- Depending on the choice during the configuration step, one section of the code will be selcted for compilation.

`core/LIS_coreMod.F90`: 
- Declare and allocate variables needed by PFIO.
- Do not finalize ESMF if PFIO is used.
- Add preprocessing directives.

`core/LIS_surfaceModelMod.F90`: 
- Include calls from the module `LIS_PFIO_historyMod.F90` for the HISTORY creation.
- Add preprocessing directives to differentiate HISTORY subroutine related calls involving PFIO or not.

`core/LIS_PRIV_rcMod.F90`: 
- Declared new variables:
    - `do_ftiming` (do code profiling?)
    - `n_vcollections` (num PFIO virtual collections)

`core/LIS_readConfigMod.F90`: 
- Added calls to read rc file variables for profiling or not, the number virtual collections (only need is PFIO used) and data compression parameters.

File modified only for profiling:
 
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

where with the default value (`0`) will generate a code that works without PFIO, i.e., the original LIS code (with not PFIO library and related calls).

In addition, the folowing configuration settings:

```bash
   NETCDF use shuffle filter? (1-yes, 0-no, default = 1): 
   NETCDF use deflate filter? (1-yes, 0-no, default = 1): 
   NETCDF use deflate level? (1 to 9-yes, 0-no, default = 9): 
```

are no longer needed because the three parameters are now set at run time in the `lis.config` file.

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
   Profiling Tool:                    yes    # do you want to profile the code (no as default)? 
   num PFIO virtual collections:      4      # number of virtual collections (1 as default)
   netCDF shuffle filter:             0
   netCDF deflate filter:             1
   netCDF deflate level:              1      # deflate level (0 as default)
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

### Model Configuration

- 5901x2801 grid points
- one-day integration with output produced every 3 hours (8 files with one record each)
- The fields to be written out are:
    - 80 2D fields
    - 4 3D fields (with 4 levels)

Without any data compression, each output file produced here requires 6.43 Gb.
Our goal here is not only to reduce the time spent on IO but also to decrease
the file size by applying data compression.

In all the experiment we did, the LIS/PFIO code produced bitwise identical results as the original version of the code.

### Results with 112 Compute Cores

We ran the orginal version of the LIS code (ORG) and the one with the PFIO implementation (PFIO) using a $14 \times 8$ processor decomposition.
As the data compression level varies, we recorded the average output file size (out of 8 files)
and the total time to complete the integration.

It is assumed that the PFIO option uses one additional node (reserved for output) with repected to the original version of the code.

| Deflation Level  | Average File Size |  | Total Time (s) |  |
|--- | ---| --- | --- | --- |
| | **ORG** | **PFIO** | **ORG** | **PFIO** |
| 0 | 6.43 | 6.43 | 734 | 1023 |
| 1 | 1.76 | 1.71 | 1484 | 1213 |
| 3 | 1.74 | 1.65 | 1928 | 1403 |
| 5 | 1.75 | 1.61 | 2121 | 1670 |
| 7 | 1.74 | 1.59 | 3388 | 2376 |
| 9 | 1.73 | 1.58 | 3948 | 8297 |


![fig_stats](fig_lis_pfio_stats.png)

### Results when the Number of Compute Cores Varies

We use the data compression level of 1 and let the number of compute cores varies. 
We record the total time for a one-day integration and the HISTORY file created every three hours.

| # Compute Cores | **ORG** | **PFIO** | Gain/Loss |
| ---- | --- | --- | --- |
| $112=14 \times  8$ | 1484 | 1213 | +18% |
| $140=14 \times 10$ | 1340 | 1198 | +11% |
| $168=14 \times 12$ | 1292 | 1272 | +1.5%|
| $196=14 \times 14$ | 1267 | 1374 | -8.4% |

As the number of computee cores increases, PFIO becomes less attractive.
It is more likely due to the fact PFIO is not done before the model completes the calculations, therefore creation data congestion in the output server node.

### Test Case 1

- 5901x2801 grid points
- Four-day integration with output produced every 1.5 hours (73 files with one record each)
- The fields to be written out are:
    - 10 2D fields
    - 2 3D fields (with 4 levels)
- Each file without data compression has a size of 412 Mb.


|            | 84      |  112    | 140     | 168    | 196    | 224    | 504     | 1008   |
|  ---       | ---:    |  ---:   | ---:    | ---:   | ---:   | ---:   | ---:    | ---:   |
| Run Method |  817.88 |  614.58 |  491.55 | 410.19 | 350.77 | 307.14 |  137.03 |  68.35 |
| Output     |  395.95 |  334.99 |  299.88 | 290.55 | 268.05 | 252.47 |  364.71 | 216.24 |
| Overall    | 1495.24 | 1224.56 | 1062.53 | 968.48 | 889.67 | 825.61 |  687.49 | 555.93 |

**Table 1.1** LIS: timiming statistics as the number of processors varies.


<!---

|            | 84      |  112    | 140     | 168    | 196    | 224    | 504     |
|  ---       | ---:    |  ---:   | ---:    | ---:   | ---:   | ---:   | ---:    |
| Run Method |  832.05 |  623.94 |  502.99 | 415.53 | 356.62 | 312.93 | 137.37 |
| Output     |   67.20 |   70.05 |   60.47 |  61.79 |  59.91 |  56.82 | 942.47 |
| Overall    | 1242.06 |  975.82 |  829.25 | 728.86 | 647.03 | 604.91 |1289.03 |

**Table 1.2** LIS/PFIO: timiming statistics as the number of processors varies and there is one IO node.

|            | 84      |  112    | 140     | 168    | 196    | 224    | 504     |
|  ---       | ---:    |  ---:   | ---:    | ---:   | ---:   | ---:   | ---:    |
| Run Method |  821.71 |  622.39 |  503.77 | 414.20 | 353.00 | 309.48 | 140.17  |
| Output     |   36.67 |   37.21 |   37.97 |  38.27 |  36.16 |  33.12 | 193.81  |
| Overall    | 1235.62 |  987.85 |  821.78 | 720.89 | 637.72 | 565.33 | 553.24  |

**Table 1.3** LIS/PFIO: timiming statistics as the number of processors varies and there are two IO nodes. There are four virtual collections and 2 backend cores per node.

--->

| # compute cores  | # IO Nodes | Run Method | Output  | Overall |
| ----             |  ----      | ----:      | ----:   | ----:   |
| 224              |  1         | 312.93     |  56.82  |  604.91 |
|                  |  2         | 309.48     |  33.12  |  565.33 |
| 504              |  1         | 137.37     | 942.47  | 1289.03 |
|                  |  2         | 140.17     | 193.81  |  553.24 |
|                  |  3         | 139.95     |  78.22  |  425.78 |
| 644              |  3         | 114.77     | 134.53  |  447.00 |
|                  |  4         |            |         |         |
| 784              |  3         |  89.97     | 284.14  |  574.70 |
|                  |  4         |            |         |         |
| 1008             |  4         |   69.97    | 217.53  |  495.42 |
|                  |  5         |   70.01    | 135.00  |  397.69 |

**Table 1.2** LIS/PFIO: timiming statistics as the number of compute processors and the number of IO node vary. In each case, we use 2 backend cores per IO nodes and set the number virtual output collections to be two times the number of IO nodes.


### Test Case 2

- 5901x2801 grid points
- one-day integration with output produced every 1.5 hours (24 files with one record each)
- The fields to be written out are:
    - 80 2D fields
    - 4 3D fields (with 4 levels)
- Each file without compression has a size of 6.43 Gb.
 
|            | 84      |  112    | 140     | 168     | 196     | 224     | 1008    |
|  ---       | ---:    |  ---:   | ---:    | ---:    | ---:    | ---:    | ---:    |
| Run Method |  373.71 |  280.62 |  224.90 |  186.62 |  160.74 |  140.67 |  31.24  |
| Output     | 1209.94 | 1229.54 | 1194.51 | 1228.61 | 1436.07 | 1420.89 | 1331.73 |
| Overall    | 2269.26 | 2171.23 | 2069.12 | 2059.28 | 2235.15 | 2188.14 | 2050.97 |

**Table 2.1** LIS: timiming statistics as the number of processors varies.


| # compute cores  | # IO Nodes | Run Method | Output  | Overall |
| ----             |  ----      | ----:      | ----:   | ----:   |
| 224              |  2         | 143.93     | 1021.35 | 1963.34 |
|                  |  3         | 144.25     |  535.86 | 1422.43 |
| 504              |  2         |  64.10     | 2106.05 | 4799.04 |
|                  |  3         |  64.07     | 1631.45 | 2358.15 |
| 1008             |  6         |  31.81     | 1341.00 | 2270.67 |
|                  |  8         |  31.81     |  603.19 | 1899.65 |

**Table 2.2** LIS/PFIO: timiming statistics as the number of compute processors and the number of IO node vary. In each case, we use 2 backend cores per IO nodes and set the number virtual output collections to be two times the number of IO nodes.

<!---

#### Comments

From the above statistics, we can make the following comments:

- Using the deflation level 1 is most cost effective in both ORG and PFIO.
    - At such level, PFIO runs faster and tends to generate a smaller file size.
- At deflation level 9, the time to collect the data and do data compression appears to be higher than the model computing time. PFIO performs very poorly.
- PFIO works better when the model computations require more time than the communications between the computing cores and the IO servers.
- As we increase the model resolution and the IO dominates the model calculations, we expect PFIO to perform better.

--->


