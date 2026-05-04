# k-Wave

Source code of the k-Wave Service on o²S²PARC. k-Wave is MATLAB/C++ toolbox for the time-domain simulation of acoustic wave fields. This Service uses the C++ function to run 3D time-domain simulation of wave propagation on a GPU using C++ CUDA code. You can find more information about k-Wave [here](http://www.k-wave.org/index.php).

The Service runs the kspaceFirstOrder-CUDA function (v1.3)

## Citing the k-Wave Toolbox
If you find the toolbox useful for your academic work, please consider citing one or more of the associated papers, more information [here](http://www.k-wave.org/license.php).

## Information for developers of the o²S²PARC Service
### Usage

```console
$ make help
$ make build
$ make publish-local 
```

### How to test
Use as input the file `validation/input/input.h5`. Running the service should generate an `output.h5` file. 
Is you provide optional boolean flags (either in the UI of in `inputs.json`), you should see additional data fields in the `output.h5` file. To inspect the output file you can use `h5ls -d output.h5`.

## Have an issue or question?

Please open an issue [in this repository](https://github.com/ITISFoundation/k-wave/issues/).

---
<p align="center">
<image src="https://github.com/ITISFoundation/osparc-simcore-python-client/blob/4e8b18494f3191d55f6692a6a605818aeeb83f95/docs/_media/mwl.png" alt="Made with love at www.z43.swiss" width="20%" />
</p>

