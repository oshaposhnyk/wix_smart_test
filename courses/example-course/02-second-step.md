_**LAST UPDATED:** 26/9/2023, by [Ran Yahalom](https://wix.slack.com/archives/D028P8YJY64)_

<!-- TOC -->
* [Project file structure](#project-file-structure)
* [Production directory (aka "production package")](#production-directory--aka--production-package--)
  * [`model.py`](#modelpy)
  * [`__init__.py`](#initpy)
  * [Conda environment file](#conda-environment-file)
    * [Tips for generating the Conda environment file](#tips-for-generating-the-conda-environment-file)
* [MLproject file](#mlproject-file)
<!-- TOC -->

# Project file structure
👉 To use the ML platform, all models should be added to the [ds-ml-models git repository](https://github.com/wix-private/ds-ml-models) as a dedicated project (sometimes referred to as a "_sub-project_" of the ds-ml-models repo.) directory which conforms to the following file structure:
```
ds-ml-models:
⋮
|-> project directory:
    |-> production directory:
        |-> conda.yaml
        |-> model.py
        |-> __init__.py 
    |-> MLproject file
    ⋮
⋮
```

👉 This is a minimal required file structure which you can extend according to your needs. For example, it is good practice to add a test folder to your project directory which contains unit tests that will be run in order to verify that everything works as expected.

# Production directory (aka "production package")
👉 This is the root directory that is packaged and stored in the model registry.

👉 All code needed for training & inference on the ML platform should be under this directory because anything outside of it will not be packaged and will therefore be missing in real-time.

👉 If your project includes non-production code for development/analysis that doesn't run on the ML platform, it should be placed outside the production directory.

## `model.py`
👉 To be able to build, deploy and invoke your model on the ML Platform, its class must inherit from `wixml.model.BaseWixModel2` ([see code](https://github.com/wix-private/data-services/blob/master/ml-framework/wix-python-ml/wixml/model/base.py)).

👉 You should define your model's class in a `model.py` located in the production directory.

👉 Without going into implementation details, here is an outline of what your model's class can look like (don't worry, we'll learn all about the methods your model is expected to implement in the ["Model Creation" lesson](https://github.com/wix-private/ds-ml-models/blob/master/ml-platform-course/02_Model_Creation/Model_creation.md)):
```python
from wixml.model import BaseWixModel2
...


class MyAmazingModel(BaseWixModel2):
    def fit(self, df, context=None, **kwargs):
        ...
    
    def predict(self, context, model_input):
        ...

    def get_training_data(self):
        ...

    def artifacts(self):
        ...

    def schema(self):
        ...

    def load_context(self, context):
        ...
```


## `__init__.py`
👉 You must also include an `__init__.py` file in your production directory for the python interpreter to recognize it as a python package (a.k.a your "production package"). 

👉 This `__init__.py` is used by the ML platform as a convention to instantiate an instance of your model when it runs the build command defined in your MLproject file (see [below](#mlproject-file)). 

👉 To do this, your `__init__.py` must include a `load_model()` function that returns an instance of your model. For example, your `__init__.py` can look like this:
```python
from production.model import MyAmazingModel

def load_model():
    return MyAmazingModel()
```

## Conda environment file
👉 Your production package must include a `conda.yaml` [file](https://conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html#creating-an-environment-from-an-environment-yml-file) that specifies your project's dependencies on 3rd party packages. 

👉 ML platform will execute your code in a Conda environment created according to this file. 

👉 Any missing dependency from this file might cause a runtime exception during training and/or inference.

👉 An example `conda.yaml` file can look like this:

```yaml
name: my-amazing-model-env
channels:
  - conda-forge
dependencies:
  - python=3.7
  - pip==23.1.2
  - pandas==1.3.5
  - scipy==1.7.3
  - pip:
      - "--index-url https://repo.dev.wixpress.com/artifactory/api/pypi/pypi-repos/simple"
      - wixml==1.3.628
```

The first line sets the name of the environment that will be created. Next, you specify the Conda channels, i.e. locations from where Conda will download the packages you specify in the `dependencies` list. In the example above, Conda will use the conda-forge channel, but you can also specify other channels you want to use. Read more about how to do this in [this guide](https://conda.io/projects/conda/en/latest/user-guide/concepts/channels.html). 

👉 Finally, if you need to use a python package that is not available in any Conda channel and can only be installed via [pip](https://pypi.org/project/pip/), you should do the following:
   1. Add the version of pip you want the package to be installed with to the dependencies list (e.g., version 23.1.2 of pip is specified in the above example). 
   2. Specify the python package that has to be installed using pip in the `pip` sub-list. This is especially relevant for Wix packages such as `wixml`, that are stored in a private [PyPi artifactory](https://repo.dev.wixpress.com/artifactory/api/pypi/pypi-local/simple/). In the above example, we tell pip to also look for packages in Wix's artifactory by providing its URL via the `--index-url` argument.

### Tips for generating the Conda environment file
👉 As a rule of thumb, if you don't have any reason to specify a certain dependency version, simply state the dependency name (`scipy` rather than `scipy==1.7.3`) and Conda will install the most updated version that fits. 

👉 If you want to set the dependency versions from your local development environment but not sure what they are, you can open a shell, activate your environment and do the following:
   - Run the [`pip show`](https://pip.pypa.io/en/stable/cli/pip_show/) command to find the version of a specific dependency. For example, `pip show scipy` will show you the version of the installed `scipy` dependency. 
   - Run the [`pip freeze`](https://pip.pypa.io/en/stable/cli/pip_freeze/) command to list the versions of all installed dependencies. 

👉 It is convenient to use the [`pipreqs`](https://pypi.org/project/pipreqs/) library to automatically generate a requirements.txt file listing the versions of **ONLY the packages your production code actually imports** (in contrast to `pip freeze` which will list **ALL** packages, regardless of whether your code actually uses them). This can be done as follows:
   1. Open a shell, activate your local development environment and install pipreqs: `pip install pipreqs`
   2. Then run the following after replacing <PROJECT_DIRECTORY> and <PRODUCTION_DIRECTORY> with the names of your project and production directories:
   ```bash
   cd ds-ml-models
   pipreqs ./<PROJECT_DIRECTORY>/<PRODUCTION_DIRECTORY> --savepath ./<PROJECT_DIRECTORY>/requirements.txt
   ```

# MLproject file 
👉 Since the ML platform is built on top of [mlflow](https://mlflow.org/), you must provide it with a text file in YAML syntax named `MLproject` (**without** any file extension). 

👉 This file provides ML platform with the following:
   - An informative name for your project. Note that this does NOT have to be identical to the name of your project directory (which is actually the "experiment name" describing your project in the [MLflow server](https://bo.wix.com/mlflow-server-stable/#/)).  
   - Which `conda.yaml` file should be used to create the Conda environment in which your project will be run.
   - The command with which to build your model.
   - The command with which to find & run your unit tests.

👉 Assuming that you want to name your project `my-amazing-model`, that your project directory is structured as described above and that your unit tests are built with the [unittest](https://docs.python.org/3/library/unittest.html) framework, your `MLproject` file should look like this:
```yaml
name: my-amazing-model

conda_env: production/conda.yaml

entry_points:
  build:
    command: 'python -c "from wixml.model import build_model; import importlib;
     production_module = importlib.import_module(\"production\");
     production_dir = production_module.__path__[0];
     model_inst = production_module.load_model();
     build_model(model_instance=model_inst, production_directory=production_dir)"'

  test:
      command: 'python -m unittest discover'
```

👉 The `build` command invokes the python interpreter via the `-c` [command line argument](https://docs.python.org/3/using/cmdline.html) to execute the following statements:
   1. Import the ML platform's `build_model()` function which fits & stores your model.
   2. Import your `__init__.py` module into `production_module`. 
   3. Use `production_module` to set `production_dir` with the full path to your `production` directory.
   4. Set `model_inst` with a new instance of your model's class, created by invoking the `load_model()` function from your `__init__.py`.
   5. Invoke the `build_model()` function on `model_inst` and `production_dir`. This will invoke your model's `BaseWixModel2.fit()` function and store it with a copy of your production directory in ML platform for future use.

You can read more about the `MLproject` file specification in [this mlflow documentation](https://www.mlflow.org/docs/latest/projects.html#mlproject-file).
