# Preface

If you are about to start developing a model interface **aaajedi** between **JEDI** and the **AAA model**, let’s assume the following prerequisites:

1. You already have an environment where **JEDI** can be successfully run.

2. You have a **JEDI build bundle**, and its CMake configuration allows you to `ecbuild_bundle()` the interface repository.

3. The **interface repository** has been initialized with the basic `/cmake` files.

# Basic Structure of the Interface Repository

`/cmake`

`/src`

​       `/aaajedi`    – contains the interface classes; this is where you will do most of your development

​​​       `/mains`      – contains `.cc` files for building application ctests

`/test`

​       `/testinput`    – YAML files used as input during ctests

​       `/main` or `/executables`     – `.cc` files for interface class ctests

​       `/testdata`     – model or observation files required when running ctests

The **interface repository** consists of two main parts: `/src` and `/test`.  

The primary development work takes place under `/src/aaajedi`.  

Each subdirectory inside `/src/aaajedi` contains one **class**, and these classes together form the foundation that connects **JEDI** with the underlying **model**.

The basic definitions of these classes can be found under  
`oops/src/oops/interface` or `oops/src/oops/base`.  However, these are only **abstract templates** or **empty skeletons** without concrete implementations.  

Our task in developing a model interface is to implement these abstract classes according to the design and requirements of our specific model.

Below is a brief description of the functionality of several fundamental classes:

- **Geometry** is the most fundamental class, providing basic information about the grid and variables.

- **GeometryIterator** iterates over the grid points.

- The **Fields** class provides basic information and operations related to model variables, which can differ significantly among models.

- **State** and **Increment** are derived from **Fields**, but each defines its own specific functions.
  
  - The **State** class represents the model state.
  
  - The **Increment** class represents the perturbation or delta relative to the state.

- Since model variables may not directly correspond to observation variables, the **VariableChange** class is developed to handle this transformation.

## Components of Each Class

Each class generally consists of the following files:  

(“+” means it will be explained in more detail later; “–” means it follows standard conventions and developers can refer to existing implementations such as **MPAS-JEDI** or **FV3-JEDI**):

1. **class_mod.F90 (+)** – the lowest-level Fortran implementation.

2. **class_interface_mod.F90 (–)** – provides Fortran functions/subroutines that can be called directly from C++ code.

3. **class.interface.h (–)** – the C++ header that exposes Fortran routines to C++ through a C++ interface.

4. **classParameters.h (–)** – reads configuration parameters from YAML files.

5. **class.cc (+)** – the C++ implementation that calls the underlying Fortran routines.

6. **class.h (–)** – the top-level C++ header that declares all class functions.

This is the **basic structure** of each class, but not every class includes all six files.  For example, `GeometryIterator` does not require `Parameters.h`.  `Fields` on the other hand, only includes a `mod.F90` file, with its implementation handled through `State` and `Increment`.

Typically, files **(1)** and **(5)** are the most critical, since the others mainly follow established conventions, while these two files vary substantially across models.  
As references, **MPAS-JEDI** and **FV3-JEDI** provide complete and well-structured examples, whereas **OOPS/qg/model** offers a simpler design that is easier to understand function by function.

# /src

## CMakeLists.txt

In the main `CMakeLists.txt`, include both the `/aaajedi` and `/mains` directories using the `add_subdirectory()` command.

## /aaajedi

### CMakeLists.txt

- **List all files** under `/aaajedi`.

- **Create the target** for building the interface.

- **Link dependencies**: include **OpenMP**, **MPI**, and other required **JEDI components**.

- **Configure include directory layout** so that the **build-tree** mirrors the **install-tree** structure.

- ***Specify Fortran module output directories** for both build and install phases of the interface.

### Traits.h

defines `namespace aaajedi { struct Traits { ... }; }`, which contains model-specific type aliases and constants used throughout the interface.

### Fortran.h

declares data types used for interoperability between Fortran and C++.





### Geometry

This is the first class to be developed, and it contains the **fundamental grid and mesh information** for the model.

#### ATLAS

In the JEDI–model interface, **ATLAS** is commonly used to handle geometry-related grid and mesh structures ([Atlas](https://sites.ecmwf.int/docs/atlas)) , a library designed for parallel data structures supporting unstructured grids and function spaces.  

ATLAS is primarily written in C++, but its main functionalities are accessible from Fortran through an F2003 interface.  It depends on **fckit** and **eckit**, which are also modeling toolkits developed by **ECMWF**.

Within the interface, ATLAS components such as `mpi_communicator`, `atlas_field`, `atlas_fieldset`, and `atlas_functionspace` are used.  The `mpi_communicator` encapsulates parallel communication operations (see https://mpitutorial.com/tutorials/).

#### geom_mod.F90

In `geometry_mod.F90`, the geometry module defines a **geometry type** `geom_type`that stores the fundamental grid-point information, the foundation for all variable-related operations.   

This module typically includes basic routines such as **`setup`**, **`clone`**, and **`delete`**. If the **ATLAS** library is used to construct the geometry, the `geom_type` can additionally include type(fckit_mpi_comm) :: f_comm and type(atlas_functionspace)  :: function_space. These components enable parallel communication and provide the underlying grid and function-space definitions used throughout the interface.

#### Geometry.cc

A key component of the `Geometry` class is its **constructor**, where the main initialization steps are implemented. The process generally includes:

1. **Reading parameters** from `GeometryParameters.h`.

2. **Initializing basic information** using the `setup` function defined in `geom_mod.F90`, based on the parameters read in step 1.  This step establishes the Fortran-side geometry object `F90geom keyGeom_` and the MPI communicator `eckit::mpi::Comm &comm_` (which corresponds to `type(fckit_mpi_comm) :: f_comm` in `geom_mod.F90`).

3. **Generating the mesh.**

The third step is the most complex and can be done in several ways:

- **Using the ATLAS `MeshGenerator`:**  
  
  This is the most straightforward approach. You only need to specify the desired grid configuration, and ATLAS automatically generates the mesh and handles parallel distribution.

- **Using the ATLAS `MeshBuilder`:**  
  
  This approach, as seen in **MPAS** or **FV3**, is more involved. You must manually prepare all mesh-related information, such as **partitions**, **global and local indices**, and **node connectivity**.  
  
  Each node must form either a **triangular** or **quadrilateral** element, and all this information must be stored and provided to the `MeshBuilder`.  This procedure is typically carried out within the `geom_get_coords_and_connectivities` function.





### GeometryIterator

responsible for looping through grid points; its implementation generally follows standard conventions.





### Fields

The **Fields**, **State**, and **Increment** classes are closely related.  The **Fields** class defines the basic information and operations for model variables, serving as the foundation for handling all variables in the model.  The **State** class represents the model’s physical state, while the **Increment** class represents the perturbation (or delta) of those variables.

In Fortran, the `State` and `Increment` types are **derived from** the `Fields` type.   Therefore, all fundamental variable-handling and arithmetic operations are implemented in the **Fields module**.  The **Fields module** itself only includes a Fortran file (`class_mod.F90`) and **cannot be directly accessed** by JEDI or OOPS.  Its functions are **overridden** by the `State` and `Increment` classes in their respective `class_interface_mod.F90` files, making them accessible to C++ through the interface layer.

#### Type Field

Each variable can first be defined as an independent **field type**.  The specific information contained in each field may vary depending on the model’s grid structure and data storage format,  but at minimum it should include the **variable name** and its **data array**.

#### Type Fields

The **Fields type** is used to store all variables and often also includes a `Geometry` type or the total number of variables.  
It can contain a wide range of functions for variable handling and operations, for example:

- Reading or writing variables from/to files

- Writing data into an **ATLAS fieldset**

- Performing arithmetic operations such as **add**, **schur product, rms**  and etc. computations





### State

The **State** class is responsible for handling **model variables** within the model interface.  It serves as the foundation for all subsequent variable operations and is therefore a **crucial component** of the interface. 

As mentioned earlier, the **State type** is derived from the **Fields type**.  Its data members can remain unchanged, but additional functions specific to the **State** class are added. For example,  `add_increment`, which adds variables from an **Increment** object to those in a **State** object.

In `State.cc`, multiple **constructors** can be defined to accommodate different ways of creating a **State** object, depending on the specific use case or initialization method.





### Increment

Its construction is similar to that of the **State** class, but it may include **additional model-specific functions**.





### VariableChange

The variables contained in a model may not always correspond directly to those required by the observations.  This class is designed to perform the **conversion between model variables and observation variables**.

Within the `VariableChange` directory, there are typically main C++ files and several subdirectories such as **`/Base`** and **`/Model2GeoVaLs`**:

- The main `VariableChange` C++ file defines the `VariableChange` class, which includes the `changeVar` function.

- The `/Base` directory contains `VariableChangeBase`, which defines the `VariableChangeBase` class, also including a `changeVar` function that provides the common interface.

- The primary development work focuses on `/Model2GeoVaLs`, where the class `VarChaModel2GeoVaLs` inherits from `VariableChangeBase`.  Its core implementation is in the Fortran subroutine `changevar` within `model2geovals_mod.F90`. This subroutine takes a **State** object as input and produces another **State** object as output, the output state is the one passed to **GeoVaLs**.  The details of this transformation depend on the specific model and observation requirements.





### Model

This class is responsible for **driving the model forward** to complete the full data assimilation cycle.

- **`setup`** establishes the initial conditions

- **`create`** or **`initialize`** runs the first step of the model setup

- **`step`** or **`propagate`** advances the model forward by one time step

For example, in **cam-jedi**, the model run is independent:

- The **`setup`** function configures relevant parameters.

- The **`initialize`** function creates the CESM case (including `./case.create`, `setup`, and `build`).

- The **`propagate`** function runs the model forward for one time step (`./case.submit`).





# /test

## CMakeList.txt

Includes all files under this directory and is used to **register the ctests**.

## executables

Contains the **executable files** used to test each class.

## testinput

Contains the **YAML configuration files** used for testing.  You can verify YAML file formatting using [https://yamlchecker.com/](https://yamlchecker.com/).

## testdata

Contains the **data files** required for running the tests.

## modelinput

Can be used to store **additional input files** for the model.
