.. only:: html

    .. note::
        :class: sphx-glr-download-link-note

        Click :ref:`here <sphx_glr_download_auto_examples_example_nn_Si.py>`     to download the full example code
    .. rst-class:: sphx-glr-example-title

    .. _sphx_glr_auto_examples_example_nn_Si.py:


.. _tut_nn:

Train a neural network potential
================================

In this tutorial, we train a neural network (NN) potential for silicon

We are going to fit the NN potential to a training set of energies and forces from
compressed and stretched diamond silicon structures (the same training set used in
:ref:`tut_kim_sw`).
Download the training set :download:`Si_training_set.tar.gz <https://raw.githubusercontent.com/mjwen/kliff/master/examples/Si_training_set.tar.gz>`
and extract the tarball: ``$ tar xzf Si_training_set.tar.gz``.
The data is stored in **extended xyz** format, and see :ref:`doc.dataset` for more
information of this format.

.. warning::
    The ``Si_training_set`` is just a toy data set for the purpose to demonstrate how to
    use KLIFF to train potentials. It should not be used to train any potential for real
    simulations.

Let's first import the modules that will be used in this example.


.. code-block:: default


    from kliff.dataset import Dataset
    from kliff.descriptors import SymmetryFunction
    from kliff.models import NeuralNetwork
    from kliff.calculators import CalculatorTorch
    from kliff import nn
    from kliff.loss import Loss









Model
-----

For a NN model, we need to specify the descriptor that transforms atomic environment
information to the fingerprints, which the NN model uses as the input. Here, we use the
symmetry functions proposed by Behler and coworkers.


.. code-block:: default


    descriptor = SymmetryFunction(
        cut_name="cos", cut_dists={"Si-Si": 5.0}, hyperparams="set30", normalize=True
    )






.. rst-class:: sphx-glr-script-out

 Out:

 .. code-block:: none

    "SymmetryFunction" descriptor initialized.




The ``cut_name`` and ``cut_dists`` tell the descriptor what type of cutoff function to
use and what the cutoff distances are. ``hyperparams`` specifies the set of
hyperparameters used in the symmetry function descriptor. If you prefer, you can provide
a dictionary of your own hyperparameters. And finally, ``normalize`` informs that the
generated fingerprints should be normalized by first subtracting the mean and then
dividing the standard deviation. This normalization typically makes it easier to
optimize NN model.

We can then build the NN model on top of the descriptor.


.. code-block:: default


    N1 = 10
    N2 = 10
    model = NeuralNetwork(descriptor)
    model.add_layers(
        # first hidden layer
        nn.Linear(descriptor.get_size(), N1),
        nn.Tanh(),
        # second hidden layer
        nn.Linear(N1, N2),
        nn.Tanh(),
        # output layer
        nn.Linear(N2, 1),
    )
    model.set_save_metadata(prefix="./kliff_saved_model", start=5, frequency=2)









In the above code, we build a NN model with an input layer, two hidden layer, and an
output layer. The ``descriptor`` carries the information of the input layer, so it is
not needed to be specified explicitly. For each hidden layer, we first do a linear
transformation using ``nn.Linear(size_in, size_out)`` (essentially carrying out :math:`y
= xW+b`, where :math:`W` is the weight matrix of size ``size_in`` by ``size_out``, and
:math:`b` is a vector of size ``size_out``. Then we apply the hyperbolic tangent
activation function ``nn.Tanh()`` to the output of the Linear layer (i.e. :math:`y`) so
as to add the nonlinearity. We use a Linear layer for the output layer as well, but
unlike the hidden layer, no activation function is applied here. The input size
``size_in`` of the first hidden layer must be the size of the descriptor, which is
obtained using ``descriptor.get_size()``. For all other layers (hidden or output), the
input size must be equal to the output size of the previous layer. The ``out_size`` of
the output layer must be 1 such that the output of the NN model gives the energy of the
atom.

The ``set_save_metadata`` function call informs where to save intermediate models during
the optimization (discussed below), and what the starting epoch and how often to save
the model.


Training set and calculator
---------------------------

The training set and the calculator are the same as explained in :ref:`tut_kim_sw`. The
only difference is that we need to use the
:mod:`~kliff.calculators.CalculatorTorch()`, which is targeted for the NN model.
Also, its ``create()`` method takes an argument ``reuse`` to inform whether to reuse the
fingerprints generated from the descriptor if it is present.


.. code-block:: default


    # training set
    dataset_name = "Si_training_set/varying_alat"
    tset = Dataset()
    tset.read(dataset_name)
    configs = tset.get_configs()

    # calculator
    calc = CalculatorTorch(model)
    calc.create(configs, reuse=True)






.. rst-class:: sphx-glr-script-out

 Out:

 .. code-block:: none

    400 configurations read from "Si_training_set/varying_alat"
    Found existing fingerprints "fingerprints.pkl".
    Reuse existing fingerprints.
    Restore mean and stdev from "fingerprints_mean_and_stdev.pkl".




Loss function
-------------

KLIFF uses a loss function to quantify the difference between the training data and
potential predictions and uses minimization algorithms to reduce the loss as much as
possible. In the following code snippet, we create a loss function that uses the
``Adam`` optimizer to minimize it. The Adam optimizer supports minimization using
`mini-batches` of data, and here we use ``100`` configurations in each minimization step
(the training set has a total of 400 configurations as can be seen above), and run
through the training set for ``10`` epochs. The learning rate ``lr`` used here is
``0.001``, and typically, one may need to play with this to find an acceptable one that
drives the loss down in a reasonable time.


.. code-block:: default


    loss = Loss(calc, residual_data={"forces_weight": 0.3})
    result = loss.minimize(method="Adam", num_epochs=10, batch_size=100, lr=0.001)






.. rst-class:: sphx-glr-script-out

 Out:

 .. code-block:: none

    Start minimization using optimization method: Adam.
    Epoch = 0       loss = 7.9039424896e+01
    Epoch = 1       loss = 7.7877323151e+01
    Epoch = 2       loss = 7.7002645493e+01
    Epoch = 3       loss = 7.6163650513e+01
    Epoch = 4       loss = 7.5341215134e+01
    Epoch = 5       loss = 7.4532175064e+01
    Epoch = 6       loss = 7.3735103607e+01
    Epoch = 7       loss = 7.2947338104e+01
    Epoch = 8       loss = 7.2166387558e+01
    Epoch = 9       loss = 7.1390523911e+01
    Epoch = 10      loss = 7.0958875656e+01
    Finish minimization using optimization method: Adam.




We can save the trained model to disk, and later can load it back if we want. We can
also write the trained model to a KIM model such that it can be used in other simulation
codes such as LAMMPS via the KIM API.


.. code-block:: default


    model.save("./final_model.pkl")
    loss.save_optimizer_stat("./optimizer_stat.pkl")

    model.write_kim_model()






.. rst-class:: sphx-glr-script-out

 Out:

 .. code-block:: none

    KLIFF trained model write to "/Users/mjwen/Applications/kliff/examples/NeuralNetwork_KLIFF__MO_000000111111_000"




.. note::
   Now we have trained an NN for a single specie Si. If you have multiple species in
   your system and want to use different parameters for different species,
   take a look at the SiC_ example.

.. _SiC: https://github.com/mjwen/kliff/blob/master/examples/eg_nn_SiC.py


.. rst-class:: sphx-glr-timing

   **Total running time of the script:** ( 0 minutes  17.861 seconds)


.. _sphx_glr_download_auto_examples_example_nn_Si.py:


.. only :: html

 .. container:: sphx-glr-footer
    :class: sphx-glr-footer-example



  .. container:: sphx-glr-download sphx-glr-download-python

     :download:`Download Python source code: example_nn_Si.py <example_nn_Si.py>`



  .. container:: sphx-glr-download sphx-glr-download-jupyter

     :download:`Download Jupyter notebook: example_nn_Si.ipynb <example_nn_Si.ipynb>`


.. only:: html

 .. rst-class:: sphx-glr-signature

    `Gallery generated by Sphinx-Gallery <https://sphinx-gallery.github.io>`_
