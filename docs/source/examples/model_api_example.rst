
Model API Example
=================

In this notebook, we'll explore some functionality of the models of this
package. We'll work with the HBV-Educational model that is implemented
in ``rrmpg.models.hbvedu``. The data we'll use, is the data that comes
with the official code release by the papers authors.

The data can be found here: http://amir.eng.uci.edu/software.php (under
"HBV Hydrological Model")

In summary we'll look at:

- How you can create one of the models
- How you can fit the model parameters to observed discharge by:

    - Using one of SciPy's global optimizer
    - Monte-Carlo-Simulation

- How you can use a fitted model to calculate the simulated discharge

.. code:: python

    # Imports and Notebook setup
    import numpy as np
    import pandas as pd
    import matplotlib.pyplot as plt

    from timeit import timeit

    from rrmpg.models import HBVEdu
    from rrmpg.tools.monte_carlo import monte_carlo
    from rrmpg.utils.metrics import nse

    # Let plots appear in the notebook
    %matplotlib notebook

First we need to load the input data into memory. There are several
files we need:

-  ``inputPrecipTemp.txt`` contains all daily input data we need
   (temperature, precipitation and month, which is a number specifying
   for each day/timestep to which month it belongs from 1 to 12)
-  ``inputMonthlyTempEvap.txt`` contains the long-term mean monthly
   temperature and potential evapotranspiration
-  ``Qobs.txt`` contains the observed discharge

Also we need some of the values specified in ``IV.txt``, like the area
of the watershed and the initial storage values. These I'll specify
directly below. Note that we don't fix the model parameter
``T_t the temperature threshold``, like the authors did, but instead
optimize this parameter as well.

.. code:: python

    daily_data = pd.read_csv('/path/to/inputPrecipTemp.txt',
                             names=['date', 'month', 'temp', 'prec'], sep='\t')
    daily_data.head()




.. raw:: html

    <div>
    <style>
        .dataframe thead tr:only-child th {
            text-align: right;
        }

        .dataframe thead th {
            text-align: left;
        }

        .dataframe tbody tr th {
            vertical-align: top;
        }
    </style>
    <table border="1" class="dataframe">
      <thead>
        <tr style="text-align: right;">
          <th></th>
          <th>date</th>
          <th>month</th>
          <th>temp</th>
          <th>prec</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <th>0</th>
          <td>1/1/1991</td>
          <td>1</td>
          <td>-1.5</td>
          <td>0.4</td>
        </tr>
        <tr>
          <th>1</th>
          <td>1/2/1991</td>
          <td>1</td>
          <td>-0.8</td>
          <td>10.5</td>
        </tr>
        <tr>
          <th>2</th>
          <td>1/3/1991</td>
          <td>1</td>
          <td>-2.8</td>
          <td>0.9</td>
        </tr>
        <tr>
          <th>3</th>
          <td>1/4/1991</td>
          <td>1</td>
          <td>-3.7</td>
          <td>4.4</td>
        </tr>
        <tr>
          <th>4</th>
          <td>1/5/1991</td>
          <td>1</td>
          <td>-6.1</td>
          <td>0.6</td>
        </tr>
      </tbody>
    </table>
    </div>



.. code:: python

    monthly = pd.read_csv('/path/to/inputMonthlyTempEvap.txt',
                          sep=' ', names=['temp', 'not_needed', 'evap'])
    monthly.head()




.. raw:: html

    <div>
    <style>
        .dataframe thead tr:only-child th {
            text-align: right;
        }

        .dataframe thead th {
            text-align: left;
        }

        .dataframe tbody tr th {
            vertical-align: top;
        }
    </style>
    <table border="1" class="dataframe">
      <thead>
        <tr style="text-align: right;">
          <th></th>
          <th>temp</th>
          <th>not_needed</th>
          <th>evap</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <th>0</th>
          <td>1.4</td>
          <td>5</td>
          <td>0.161</td>
        </tr>
        <tr>
          <th>1</th>
          <td>-0.3</td>
          <td>5</td>
          <td>0.179</td>
        </tr>
        <tr>
          <th>2</th>
          <td>2.6</td>
          <td>20</td>
          <td>0.645</td>
        </tr>
        <tr>
          <th>3</th>
          <td>6.3</td>
          <td>50</td>
          <td>1.667</td>
        </tr>
        <tr>
          <th>4</th>
          <td>10.9</td>
          <td>95</td>
          <td>3.065</td>
        </tr>
      </tbody>
    </table>
    </div>



.. code:: python

    qobs = pd.read_csv('/path/to/Qobs.txt',
                       names=['values'])
    qobs.head()




.. raw:: html

    <div>
    <style>
        .dataframe thead tr:only-child th {
            text-align: right;
        }

        .dataframe thead th {
            text-align: left;
        }

        .dataframe tbody tr th {
            vertical-align: top;
        }
    </style>
    <table border="1" class="dataframe">
      <thead>
        <tr style="text-align: right;">
          <th></th>
          <th>values</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <th>0</th>
          <td>4.5</td>
        </tr>
        <tr>
          <th>1</th>
          <td>11.0</td>
        </tr>
        <tr>
          <th>2</th>
          <td>6.6</td>
        </tr>
        <tr>
          <th>3</th>
          <td>5.0</td>
        </tr>
        <tr>
          <th>4</th>
          <td>4.1</td>
        </tr>
      </tbody>
    </table>
    </div>



.. code:: python

    # Values taken from the IV.txt
    area = 410
    soil_init = 100
    s1_init = 3
    s2_init = 10

Create a model
--------------

Now let's have a look how we can create one of the models implemented in
``rrmpg.models``. Basically for all models we have two different
options: 1. Initialize a model with all mandatory inputs but **without**
specific model parameters. 2. Initialize a model with all mandatory
inputs **with** specific model parameters.

In the `documentation <http://rrmpg.readthedocs.io>`__ you can find a
list of all model parameters or you can look at help(HBVEdu) for this
model. In the case you don't provide specific model parameters, random
parameters will be generated that are in between the default parameter
bounds. You can have a look at these bounds by calling
get\_param\_bounds() on a model.

For now we don't know any specific parameter values, so we'll create on
with random parameters. We only need to specify the watershed area for
this model.

.. code:: python

    model = HBVEdu(area=area)

Fit the model to observed discharge
-----------------------------------

As already said above, we'll look at two different methods implemented
in this package: 1. Using one of SciPy's global optimizer 2.
Monte-Carlo-Simulation

Using one of SciPy's global optimizer
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Each model has a ``.fit()`` method. This function uses the globel
optimizer `differential
evolution <https://docs.scipy.org/doc/scipy/reference/generated/scipy.optimize.differential_evolution.html>`__
from the scipy package to find the set of model parameters that produce
the best simulation, regarding the provided observed discharge array.
The inputs for this function can be found in the
`documentation <http://rrmpg.readthedocs.io>`__ or the help().

.. code:: python

    help(model.fit)


.. code-block:: none

    Help on method fit in module rrmpg.models.hbvedu:

    fit(qobs, temp, prec, month, PE_m, T_m, snow_init=0.0, soil_init=0.0, s1_init=0.0, s2_init=0.0) method of rrmpg.models.hbvedu.HBVEdu instance
        Fit the HBVEdu model to a timeseries of discharge.

        This functions uses scipy's global optimizer (differential evolution)
        to find a good set of parameters for the model, so that the observed
        discharge is simulated as good as possible.

        Args:
            qobs: Array of observed streamflow discharge.
            temp: Array of (mean) temperature for each timestep.
            prec: Array of (summed) precipitation for each timestep.
            month: Array of integers indicating for each timestep to which
                month it belongs [1,2, ..., 12]. Used for adjusted
                potential evapotranspiration.
            PE_m: long-term mean monthly potential evapotranspiration.
            T_m: long-term mean monthly temperature.
            snow_init: (optional) Initial state of the snow reservoir.
            soil_init: (optional) Initial state of the soil reservoir.
            s1_init: (optional) Initial state of the near surface flow
                reservoir.
            s2_init: (optional) Initial state of the base flow reservoir.

        Returns:
            res: A scipy OptimizeResult class object.

        Raises:
            ValueError: If one of the inputs contains invalid values.
            TypeError: If one of the inputs has an incorrect datatype.
            RuntimeErrror: If the monthly arrays are not of size 12 or there
                is a size mismatch between precipitation, temperature and the
                month array.



.. code:: python

    # We don't have an initial value for the snow storage,  so we omit this input
    result = model.fit(qobs.values, daily_data.temp, daily_data.prec,
                       daily_data.month, monthly.evap, monthly.temp,
                       soil_init=soil_init, s1_init=s1_init,
                       s2_init=s2_init)

``result`` is an object defined by the scipy library and contains the
optimized model parameter, as well as some more information on the
optimization prozess. Let us have a look on this object

.. code:: python

    result




.. parsed-literal::

         fun: 9.3994864331708197
     message: 'Optimization terminated successfully.'
        nfev: 7809
         nit: 44
     success: True
           x: array([ -2.97714592e-01,   3.77549145e+00,   1.99253759e+02,
             1.59770799e+00,   1.16985627e-02,   9.16944196e+01,
             5.03153295e-02,   7.93555552e-02,   1.02132677e-02,
             4.93546960e-02,   4.57495030e+00])



Some of the relevant informations are:

- ``fun`` is the final value of our optimization criterion, the mean-squared-error in this case
- ``message`` describes the cause of the optimization termination
- ``nfev`` is the number of model simulations
- ``sucess`` is a flag wether or not the optimization was successful
- ``x`` are the optimized model parameters

Now let's set the model parameter to the optimized parameter found by
the optimizer. Therefore we need to create a dictonary containing on key
for each model parameter and as the corresponding value the optimized
parameter. The list of model parameter names can be retrieved by the
``model.get_parameter_names()`` function. We can then create the needed
dictonary by the following lines of code.

.. code:: python

    params = {}

    param_names = model.get_parameter_names()

    for i, param in enumerate(param_names):
        params[param] = result.x[i]

    # This line set the model parameters to the ones specified in the dict
    model.set_params(params)

    # To be sure, let's look at the current model parameters
    model.get_params()




.. parsed-literal::

    {'Beta': 1.5977079902649263,
     'C': 0.011698562668775968,
     'DD': 3.775491450515529,
     'FC': 199.25375851504469,
     'K_0': 0.05031532948317842,
     'K_1': 0.079355555184944859,
     'K_2': 0.010213267703180977,
     'K_p': 0.049354696028968914,
     'L': 4.5749503030552034,
     'PWP': 91.694419598406654,
     'T_t': -0.29771459243043052}



Also it might not be clear at the first look, this are the same
parameters as the ones specified in ``result.x``. In ``result.x`` they
are ordered according to the ordering of the ``_param_list`` specified
in each model class, where ass the dictonary output here is
alphabetically sorted.

Monte-Carlo-Simulation
^^^^^^^^^^^^^^^^^^^^^^

Now let us have a look how we can use the Monte-Carlo-Simulation
implemented in ``rrmpg.tools.monte_carlo``.

.. code:: python

    help(monte_carlo)


.. code-block:: none

    Help on function monte_carlo in module rrmpg.tools.monte_carlo:

    monte_carlo(model, num, qobs=None, **kwargs)
        Perform Monte-Carlo-Simulation.

        This function performs a Monte-Carlo-Simulation for any given hydrological
        model of this repository.

        Args:
            model: Any instance of a hydrological model of this repository.
            num: Number of simulations.
            qobs: (optional) Array of observed streamflow.
            **kwargs: Keyword arguments, matching the inputs the model needs to
                perform a simulation (e.g. qobs, precipitation, temperature etc.).
                See help(model.simulate) for model input requirements.

        Returns:
            A dictonary containing the following two keys ['params', 'qsim']. The
            key 'params' contains a numpy array with the model parameter of each
            simulation. 'qsim' is a 2D numpy array with the simulated streamflow
            for each simulation. If an array of observed streamflow is provided,
            one additional key is returned in the dictonary, being 'mse'. This key
            contains an array of the mean-squared-error for each simulation.

        Raises:
            ValueError: If any input contains invalid values.
            TypeError: If any of the inputs has a wrong datatype.



As specified in the help text, all model inputs needed for a simulation
must be provided as keyword arguments. The keywords need to match the
names specified in the ``model.simulate()`` function. We'll create a new
model instance and see how this works for the HBVEdu model.

.. code:: python

    model2 = HBVEdu(area=area)

    # Let use run MC for 1000 runs, which is in the same range as the above optimizer
    result_mc = monte_carlo(model2, num=10000, qobs=qobs.values,
                            temp=daily_data.temp, prec=daily_data.prec,
                            month=daily_data.month, PE_m=monthly.evap,
                            T_m=monthly.temp, soil_init=soil_init,
                            s1_init=s1_init, s2_init=s2_init)

    # Get the index of the best fit (smallest mean squared error)
    idx = np.argmin(result_mc['mse'][~np.isnan(result_mc['mse'])])

    # Get the optimal parameters and set them as model parameters
    optim_params = result_mc['params'][idx]

    params = {}

    for i, param in enumerate(param_names):
        params[param] = optim_params[i]

    # This line set the model parameters to the ones specified in the dict
    model2.set_params(params)

Calculate simulated discharge
-----------------------------

Now that we have two models, optimized by different methods, let's
calculate the simulated streamflow of each model and compare the
results. Each model has a ``.simulate()`` method, that returns the
simulated discharge for the inputs we provide to this function.

.. code:: python

    # simulated discharge of the model optimized by the .fit() function
    qsim_fit = model.simulate(daily_data.temp, daily_data.prec,
                              daily_data.month, monthly.evap, monthly.temp,
                              soil_init=soil_init, s1_init=s1_init,
                              s2_init=s2_init)

    # simulated discharge of the model optimized by monte-carlo-sim
    qsim_mc = model2.simulate(daily_data.temp, daily_data.prec,
                              daily_data.month, monthly.evap, monthly.temp,
                              soil_init=soil_init, s1_init=s1_init,
                              s2_init=s2_init)

    # Calculate and print the Nash-Sutcliff-Efficiency for both simulations
    nse_fit = nse(qobs.values, qsim_fit)
    nse_mc = nse(qobs.values, qsim_mc)

    print("NSE of the .fit() optimization: {:.4f}".format(nse_fit))
    print("NSE of the Monte-Carlo-Simulation: {:.4f}".format(nse_mc))


.. parsed-literal::

    NSE of the .fit() optimization: 0.6170
    NSE of the Monte-Carlo-Simulation: 0.5283


And let us finally have a look at some window of the simulated
timeseries compared to the observed discharge

.. code:: python

    # Plot last year of the simulation
    plt.plot(qobs.values[-366:], color='cornflowerblue', label='Qobs')
    plt.plot(qsim_fit[-365:], color='coral', label='Qsim .fit()')
    plt.plot(qsim_mc[-365:], color='mediumseagreen', label='Qsim mc')
    plt.legend()


.. raw:: html

    <img src="data:image/png;base64,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" width="781">



Albeit not perfect, I guess it's not bad either for an out-of-the-box
optimization framework that works for any model. And since this package
is also about speed, let us see how long it takes to optimize the
HBV-Educational model performing a Monte-Carlo-Simulation for 10,000
runs.

.. code:: python

    %%timeit
    monte_carlo(model2, num=10000, qobs=qobs.values, temp=daily_data.temp,
                prec=daily_data.prec, month=daily_data.month, PE_m=monthly.evap,
                T_m=monthly.temp, soil_init=soil_init, s1_init=s1_init,
                s2_init=s2_init)


.. parsed-literal::

    1.68 s ± 29.6 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
