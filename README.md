# Theano-MPI
Theano-MPI is a python framework for distributed training of deep learning models built in Theano. It implements data-parallelism in serveral ways, e.g., Bulk Synchronous Parallel, [Elastic Averaging SGD](https://arxiv.org/abs/1412.6651) and [Gossip SGD](https://arxiv.org/abs/1611.09726). This project is an extension to [theano_alexnet](https://github.com/uoguelph-mlrg/theano_alexnet), aiming to scale up the training framework to more than 8 GPUs and across nodes. Please take a look at this [technical report](http://arxiv.org/abs/1605.08325) for an overview of implementation details. To cite our work, please use the following bibtex entry.

```bibtex
@article{ma2016theano,
  title = {Theano-MPI: a Theano-based Distributed Training Framework},
  author = {Ma, He and Mao, Fei and Taylor, Graham~W.},
  journal = {arXiv preprint arXiv:1605.08325},
  year = {2016}
}
```

Theano-MPI is compatible for training models built in different framework libraries, e.g., [Lasagne](https://github.com/Lasagne/Lasagne), [Keras](https://github.com/fchollet/keras), [Blocks](https://github.com/mila-udem/blocks), as long as its model parameters can be exposed as theano shared variables. Theano-MPI also comes with a light-weight layer library for you to build customized models. See [wiki](https://github.com/uoguelph-mlrg/Theano-MPI/wiki) for a quick guide on building customized neural networks based on them. Check out the examples of building [Lasagne VGGNet](https://github.com/uoguelph-mlrg/Theano-MPI/blob/master/theanompi/models/lasagne_model_zoo/vgg16.py), [Wasserstein GAN](https://github.com/uoguelph-mlrg/Theano-MPI/blob/master/theanompi/models/lasagne_model_zoo/wgan.py), [LS-GAN](https://github.com/uoguelph-mlrg/Theano-MPI/blob/master/theanompi/models/lasagne_model_zoo/lsgan.py) and [Keras Wide-ResNet](https://github.com/uoguelph-mlrg/Theano-MPI/tree/master/theanompi/models/keras_model_zoo/wresnet.py).

## Dependencies

Theano-MPI depends on the following libraries and packages. We provide some guidance to the installing them in [wiki](https://github.com/uoguelph-mlrg/Theano-MPI/wiki/Install-dependencies-of-Theano-MPI).
* [OpenMPI](http://www.open-mpi.org/) 1.8 + or an MPI-2 standard equivalent that supports CUDA.
* [mpi4py](https://pypi.python.org/pypi/mpi4py) built on OpenMPI.
* [numpy](http://www.numpy.org/)
* [Theano](http://deeplearning.net/software/theano/) 0.9 +
* [zeromq](http://zeromq.org/bindings:python)
* [hickle](https://github.com/telegraphic/hickle)
* [CUDA](https://developer.nvidia.com/cuda-toolkit-70) 7.5 +
* [cuDNN](https://developer.nvidia.com/cudnn) a version compatible with your CUDA Installation.
* [pygpu](http://deeplearning.net/software/libgpuarray/installation.html)
* [NCCL](https://github.com/NVIDIA/nccl)

## Installation 

Once all dependeices are ready, one can clone Theano-MPI and install it by the following.

```
 $ python setup.py install [--user]
```

## Usage

To accelerate the training of Theano models in a distributed way, Theano-MPI tries to identify two components:

* the iterative update function of the Theano model
* the parameter sharing rule between instances of the Theano model


It is recommended to organize your model and data definition in the following way.

  * `launch_session.py` or `launch_session.cfg`
  * `models/*.py`
    * `__init__.py`
    * `modelfile.py` : defines your customized ModelClass
    * `data/*.py`
      * `dataname.py` : defines your customized DataClass

Your ModelClass in `modelfile.py` should at least have the following attributes and methods:

* `self.params` : a list of Theano shared variables, i.e. trainable model parameters
* `self.data` : an instance of your customized DataClass defined in `dataname.py`
* `self.compile_iter_fns` : a method, your way of compiling train_iter_fn and val_iter_fn
* `self.train_iter` : a method, your way of using your train_iter_fn
* `self.val_iter` : a method, your way of using your val_iter_fn
* `self.adjust_hyperp` : a method, your way of adjusting hyperparameters, e.g., learning rate.
* `self.cleanup` : a method, necessary model and data clean-up steps.

Your DataClass in `dataname.py` should at least have the following attributes:

* `self.n_batch_train` : an integer, the amount of training batches needed to go through in an epoch
* `self.n_batch_val` : an integer, the amount of validation batches needed to go through during validation

After your model definition is complete, you can choose the desired way of sharing parameters among model instances:

* BSP (Bulk Syncrhonous Parallel)
* EASGD (Elastic Averaging SGD)
* GOSGD (Gossip SGD)

Below is an example launch config file for training a customized ModelClass on two GPUs.

```bash
# launch_session.cfg
RULE=BSP
MODELFILE=models.modelfile
MODELCLASS=ModelClass
DEVICES=cuda0,cuda1
```
Then you can launch the training session by calling the following command:

```bash
 $ tmlauncher -cfg=launch_session.cfg
```

Alternatively, you can launch sessions within python as shown below:

```python
# launch_session.py
from theanompi import BSP

rule=BSP()
# modelfile: the relative path to the model file
# modelclass: the class name of the model to be imported from that file
rule.init(devices=['cuda0', 'cuda1'] , 
          modelfile = 'models.modelfile', 
          modelclass = 'ModelClass') 
rule.wait()
```
More examples can be found [here](https://github.com/uoguelph-mlrg/Theano-MPI/tree/master/examples).

## Example Performance

###BSP tested on up to eight Tesla K80 GPUs
Time per 5120 images in seconds: [allow_gc = True]

| Model | 1GPU  | 2GPU  | 4GPU  | 8GPU  |
| :---: | :---: | :---: | :---: | :---: |
| AlexNet-128b | 20.42 | 10.68 | 5.45 | 3.11 |
| GoogLeNet-32b | 73.24 | 35.88 | 18.45 | 9.59 |
| VGGNet-16b | 353.48 | 191.97 | 99.18 | 66.89 |
| VGGNet-32b | 332.32 | 163.70 | 82.65 | 49.42 |
<img src=https://github.com/uoguelph-mlrg/Parallel-training/raw/master/show/val_a.png width=500/>
<img src=https://github.com/uoguelph-mlrg/Parallel-training/raw/master/show/val_g.png width=500/>

## Note

* To get the best running speed performance, the memory cache may need to be cleaned before running.

* Binding cores according to your NUMA topology may give better performance. Try the `-bind` option with the launcher.

* Learnining rate and other hyperparams may need to be retuned according to number of workers and effective batch size to be stable and give optimal convergence. 

* Shuffling training examples before asynchronous training makes the loss surface a lot smoother during model converging.

* Some known bugs and possible enhancement are listed in [Issues](https://github.com/uoguelph-mlrg/Theano-MPI/issues). We welcome all kinds of participation (bug reporting, discussion, pull request, etc) in improving the framework.

## License

© Contributors, 2016-2017. Licensed under an [ECL-2.0](https://github.com/uoguelph-mlrg/Theano-MPI/blob/master/LICENSE) license.
