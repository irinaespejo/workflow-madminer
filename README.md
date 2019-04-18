[![Build status](https://travis-ci.com/irinaespejo/workflow-madminer.svg?branch=master)](https://travis-ci.org/irinaespejo)


# Madminer deployment using yadage and docker containerization

## About
This repository is a  is to create a full deployment of [MadMiner](https://github.com/johannbrehmer/madminer) (by Johann Brehmer, Felix Kling, and Kyle Cranmer) that parallelizes its bottlenecks, is reproducible and it facilitates the use of the tool in the community. 
To achieve this we have generated a workflow using [yadage](https://github.com/yadage/yadage)
(by Lukas Heinrich) and a containerization of the software dependencies in several docker images.

This repo includes the workflow for the physics processing (config, generation of events with MadGraph, Delphes) and the machine learning processing (configuration, sampling, training) in a modular way. This means that each of this parts has its own workflow setup so that the user can mix-match. For instance, once the physics processes are run, one can play with different hyperparameters or samplings in the machine learning part without having to re-run MadGraph again.

Please, refer to the links for more information and tutorials about [MadMiner](https://madminer.readthedocs.io/en/latest/index.html) [tutorial] (https://github.com/johannbrehmer/madminer/blob/master/examples/tutorial/tutorial_1.ipynb) and [yadage](https://yadage.readthedocs.io/en/latest/) [tutorial](https://yadage.github.io/tutorial/)


## Docker images
MadMiner is a set of complex tools with many steps and for that reason we considered better to split the software dependencies and the code for the workflow in two docker images plus another one containing the latest MadMiner library version. All of the official images are hosted in the [madminertool](https://cloud.docker.com/u/madminertool/repository/list) DockerHub.

- [madminertool/docker-madminer](https://cloud.docker.com/u/madminertool/repository/docker/madminertool/docker-madminer)
Contains only latest version of MadMiner
- [madminertool/docker-madminer-phyics](https://cloud.docker.com/u/madminertool/repository/docker/madminertool/docker-madminer-physics)
Contains the code necessary to configure, generate and process events according to MadMiner. You will also find the software dependencies in the directory `/home/software`
- [madminertool/docker-madminer-ml](https://cloud.docker.com/u/madminertool/repository/docker/madminertool/docker-madminer-ml)
Contains the code necessary to configure, train and evaluate in the MadMiner framework.

To pull any of the images and see its content

```bash
$ docker pull madminertool/<image-name>
$ docker run -it madminertool/<image-name> bash
<container-id>#/home $ ls
```
If you want to check the Dockerfile for the last two images go to `worklow-madminer/docker`.

*The point of this repository is to make the life easy for the user so you won't need to figure out yourself the arguments of the scripts on `/home/code/` nor how to input new observables. The whole pipeline will be automatically generated when you follow the steps in the sections below. Also you will have the chance to input your own parameters, observables, cuts etc. without even needing to pull yourself  the docker images.*

## Install the dependencies and run the workflows
Installing the dependiencies depends on how you want to run the workflow: locally using yadage alone or in a REANA cluster.

### Local yadage-pure deployment

Install yadage and the dependencies to visualize workflows [viz]
```bash
  pip install yadage[viz]
```
Also, you need the graphviz package
Check it was succesful by running:
```bash
  yadage-run -t from-github/testing/local-helloworld workdir workflow.yml -p par=World
```
It should output lines similar to this one `2019-01-11 09:51:51,601 |         yadage.utils |   INFO | setting up backend multiproc:auto with opts {}` and no errors. Refer to the yadage links above for more details.


#### Deploy with yadage
For the first run we recommend using our default files `input.yml` and `input_delphes.yml`.
To generate the following workflow 

![image of the workflow](images/workflow-physics.png)
![image of the workflow](images/workflow-ml.png)

run 
```bash
  $ yadage-run workdir workflow-physics/workflow.yml -p inputfile='"workflow-physics/input.yml"' -p njobs="10" -p inputdelphes='"workflow-physics/input_delphes.yml"' -d initdir=$PWD --visualize
  $ rm -rf workdir/
  $ yadage-run workdir workflow-ml/workflow.yml -p inputfile='"workflow-ml/inputs/input_ML.yaml"' -p ntrainsamples="5" -p combinedfile='"workflow-ml/inputs/madminer_example_with_data_1.h5"' -d initdir=$PWD --visualize
```
to run again the command you must first remove workdir `rm -rf workdir/`
>what is every element in the command?
	- `workdir` is creating a new dir where all the intermediate and output files will be saved.
	- `workflow.yml` is the file that connects the different stages of the workflow, it must be placed in the working directory
	- all the parameters are preceed by `-p`: `njobs` is the number of maps in the workflow, `inputfile` has the parameters and `input_delphes.yml` for observables and cuts.
	- `-d initdir=$PWD` initializes the workflow in the present directory
	- `--visualize` generates an image of the workflow
	
### Local REANA deployment

_This is work in progress_


To deploy Madminer locally using [REANA](http://www.reana.io/) we will use Minikube as emulator for a cluster. Here we will give basic steps to set up Minikube as a `reana-cluster`. Please refer to https://reana-cluster.readthedocs.io/en/latest/gettingstarted.html  for more details. You will need to have [Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/) v0.28.2 and [Kubernettes](https://kubernetes.io/docs/tasks/tools/install-kubectl/) v1.11.2  installed.

Run (`-vm--driver`  may depend on the OS)

```bash
$ minikube start --kubernetes-version="v1.11.2" --vm-driver kvm2

$ virtualenv ~/.virtualenvs/myreana

$ source ~/.virtualenvs/myreana/bin/activate

(myreana) $ pip install reana-cluster

(myreana) $ reana-cluster init

(myreana) $ reana-cluster status

(myreana) $ eval $(reana-cluster env --include-admin-token)
```
with this you have set up a local `reana-cluster`. Now let's set up the` reana-client` where you will run the analysis. Please refer to https://reana-client.readthedocs.io/en/latest/gettingstarted.html  for more details.

```bash
(myreana) $ pip install reana-client
```
check connectivity to `reana-cluster`
```bash
(myreana) $ reana-client ping
```

Create the analysis

```bash
(myreana) $ reana-client create -n my-analysis

(myreana) $ export REANA_WORKON=my-analysis

(myreana) $ reana-client upload ./data

(myreana) $ reana-client start

(myreana) $ reana-client status
```
it might take a long time to finish, once it does download files
```bash
(myreana) $ reana-client download <path/to/file/on/reana/workon>
```




## Analysis structure
### 1. Analysis code
#### Physics part
configurate.py - Put together inputs and initialize
	- Initializes MadMiner, add parameters, add benchmarks, set benchmarks from morphing and save.
	`python code/configurate.py` 
	
generate.py - Generate events scripts
	- Prepare scripts for MadGraph for background and signal events based on previous optimization.
	`python code/generate.py {njobs} {h5_file}` where `{njobs}` is the initial parameter  `njobs` and `{h5_file}` is a file generated in configurate.py with the MadMiner configuration.

delphes.py - Run Delphes
	- Pass the events through Delphes, add observables and cuts, save.
	 `python code/delphes.py {h5_file} {event_file} {input_file}` where  `{h5_file}` is the same file as above  `{event_file}` is the file  `tag_1_pythia8_events.hepmc.gz` and `{input_file}{input_file}` is the initial `input_delphes.yml`

#### ML part

### 2. Anaylsis workflow
Without taking into account the inputs ans the map-reduce the general strcture of the workflow is the following

				+--------------+
				|  Configurate |
				+--------------+
					   |
					   |
					   v
				+--------------+
				|   Generate   |
				+--------------+
	   				   |
					   |
					   v
				+--------------+
				|   MG+Pythia  |
				+--------------+
					   |
					   |
					   v
				+--------------+
				|   Delphes    |
				+--------------+
					   |
					   |
					   v
				+-----------------+
				|   Sampling      |
				+-----------------+
				           |
					   |
					   v
                                +-----------------+
				|   Training      |
				+-----------------+
				           |
					   |
					   v
                                +-----------------+
				|    Testing      |
				+-----------------+

