---
marp: true
paginate:  true
theme: portrait
class: invert
---

# Spack Environments

## Programmer Meeting

### November 17, 2022

**Michael Jähn, C2SM**

---

# Environments in Spack

## Overview

- allows to work with independent groups of packages separately
- reproducible
- similar to virtual environments in other systems (e.g., python, conda)
- based around file formats: `spack.yaml`, `spack.lock` → sharable

---

# Environments in Spack

## Features

- establish standard software requirements for your project(s)
- set up run environments for users
- support your usual development environment(s)
- set up packages for CI/CD
- reproduce builds (approximately or exactly) on other machines

---

# Environments in Spack

## Basics

Let’s look at the output of `spack find`:

```shell
$ spack find
==> 18 installed packages
-- cray-cnl7-haswell / gcc@8.3.0 --------------------------------
boost@1.79.0                cosmo-dycore@c2sm-features  slurm@20.11.8
cosmo-dycore@c2sm-features  cuda@11.0

-- cray-cnl7-haswell / nvhpc@21.3 -------------------------------
cmake@3.17.0                        mpich@7.7.18
cosmo@c2sm-features                 nasm@2.15.05
cosmo-eccodes-definitions@2.19.0.5  netcdf-c@4.7.4.4
eccodes@2.19.1                      netcdf-fortran@4.5.3
jasper@1.900.1                      openjpeg@2.3.1
libgrib1@22-01-2020                 perl@5.26.1
libjpeg-turbo@2.1.3
```

- complete, but cluttered list
- packages with different compilers
- potentially packages built with different `mpi` libraries
- multiple variants

---

# Environments in Spack

## Creating and activating environments

Create an environment:

```shell
$ spack env create myproject
==> Created environment 'myproject' in 
    /home/spack/spack/var/spack/environments/myproject
==> You can activate this environment with:
==>   spack env activate myproject
``` 

See created environments:

```shell
$ spack env list
==> 1 environments
    myproject
``` 

---

# Environments in Spack

## Creating and activating environments

Now let’s **activate** our environment:

```shell
spack env activate myproject # spacktivate myproject
```

Once activated, `spack find` only shows what is in the current environment. 

```shell
$ spack find
==> In environment myproject
==> No root specs
==> 0 installed packages
``` 

---

# Environments in Spack

## Creating and activating environments

To check what environment we are in:

```shell
$ spack env status
==> In environment myproject
```

Leaving an environment:

```shell
$ despacktivate  # short alias for `spack env deactivate`"
$ spack env status
==> No active environment
$ spack find
``` 

Quiz: What will `spack find` show now?

---

# Environments in Spack

## Creating and activating environments

After deactivating, we can see everything installed in this Spack instance:

```shell
$ despacktivate  # short alias for `spack env deactivate`"
$ spack env status
==> No active environment
$ spack find
==> 18 installed packages
-- cray-cnl7-haswell / gcc@8.3.0 --------------------------------
boost@1.79.0                cosmo-dycore@c2sm-features  slurm@20.11.8
cosmo-dycore@c2sm-features  cuda@11.0

-- cray-cnl7-haswell / nvhpc@21.3 -------------------------------
cmake@3.17.0                        mpich@7.7.18
cosmo@c2sm-features                 nasm@2.15.05
cosmo-eccodes-definitions@2.19.0.5  netcdf-c@4.7.4.4
eccodes@2.19.1                      netcdf-fortran@4.5.3
jasper@1.900.1                      openjpeg@2.3.1
libgrib1@22-01-2020                 perl@5.26.1
libjpeg-turbo@2.1.3
```

---

# Using Environments with COSMO 

`cosmo-cpu.yaml`:

```yaml
spack:
    specs:
        - cosmo@c2sm-features%nvhpc@21.3 cosmo_target=cpu ~cppdycore +eccodes
        - cosmo-eccodes-definitions@2.19.0.5
        - eccodes@2.19.1 +fortran
        - libgrib1@22-01-2020
    concretizer:
        unify: true
```

`cosmo-gpu.yaml`:
```yaml
spack:
    specs:
        - cosmo@c2sm-features%nvhpc@21.3 cosmo_target=gpu +cppdycore +eccodes
          # ~build_tests, otherwise --test=root will trigger Dycore tests as well
        - cosmo-dycore@c2sm-features%gcc@8.3.0 ~build_tests
          # needs to be a root spec, otherwise we have twice (%gcc and %nvhpc)
          # this leads to a concretize-error in test_cosmo.py
        - cmake%nvhpc
        - cosmo-eccodes-definitions@2.19.0.5
        - eccodes@2.19.1 +fortran
        - libgrib1@22-01-2020
          # if not %nvhpc, spack links against %gcc (coming from Dycore)
          # this is not possible for the .mod files needed to match nvhpc
        - mpich%nvhpc
    concretizer:
        unify: true
```

---
# Using Environments with COSMO 

## Workflow with `spack install`

```shell
spack env create cosmo-gpu cosmo-gpu.yaml
spacktivate -p cosmo-gpu # spack env activate -p cosmo-gpu
spack concretize
spack install --test=root
```

---
# Using Environments with COSMO 

## Spack development workflow with `spack develop`

We follow the approach from the official [spack documentation on developer workflows](https://spack-tutorial.readthedocs.io/en/latest/tutorial_developer_workflows.html).

```shell
spack env create -d cosmo-gpu_dev cosmo-gpu.yaml
cd cosmo-gpu_dev
spacktivate -p .
```

---

Now, we use `spack install` to build the entire development tree:

```shell
spack install
```

Afterwards, we specify which package and version we want to work on.

```shell
spack develop cosmo-dycore@c2sm-features
```

This has now been added to our spack.yaml:

```shell
grep -3 develop: spack.yaml
```

---

Then, we have to re-concretize and re-build.

```shell
spack concretize -f
spack install
```

This rebuilds `cosmo-dycore` from its subdirectory. 

Now, we can make changes to `cosmo-dycore`
and re-build with `spack install`. 

To work on other packages of the spec, start again from
`spack develop <package>@variant`.

---

### Open Issues

- Combine `cosmo-cpu.yaml` and `cosmo-gpu.yaml` and  into a single file

