.. _Testing:

==============================
Testing the FlightDataAnalyser
==============================

----------------------------------
Patching settings Values for tests
----------------------------------

Sometimes it can be easier to change the value of one of the settings than
having to generate long or complicated datasets in order to suit the required
setting. Clearly to keep the code realistic, one should still check the
setting works - ideally by not patching it in one test or by patching it with
a value above and below the threshold used in the code.

Here's how to patch the setting using `mock.patch` where the module imports
analysis_engine.settings like so:

    # my_module.py
    import analysis_engine.settings as settings

.. code-block:: python
    :linenos:
    
    # my_module_test.py
    import mock
    
    class TestSomething(unittest.TestCase):
        @mock.patch("analysis_engine.settings.VALUE_TO_REPLACE_WITH_NEW", new=10)
        def test_something(self):
            ....
            
However be aware that if settings are imported directly into the module, the 
variable now belongs in that module so the patch path needs to reflect this:

    # my_module.py
    from analysis_engine.settings import VALUE_TO_REPLACE_WITH_NEW
    
.. code-block:: python
    :linenos:

    # my_module_test.py
    class TestSomething(unittest.TestCase):
        @mock.patch("analysis_engine.my_module.VALUE_TO_REPLACE_WITH_NEW", new=10)
        def test_something(self):
            ....
    

----------------------------------
Preparing a Stripped Down HDF File
----------------------------------

To improve the AnalysisEngine test suite we should create assertions against
real data. Since nodes may require a complex dependency tree, creating nodes
and loading multiple numpy arrays from .npy files is a very slow process. It is
much easier to create tests against HDF files instead.

Since we only require a subset of parameters within an HDF file for our tests,
files should be stripped of unnecessary parameters to avoid storing large
data files within revision control. There are two different utilities designed
for this purpose.

 * :ref:`trimmer` - Strips an hdf file of everything but the dependencies of a specified list of nodes.
 * strip - Strips an hdf file of everything but a specified list of parameters.

There is a utility within the HDFAccess repository designed for this purpose.
The following command will display help information::

    # python HDFAccess/hdfaccess/utils.py strip -h

    usage: utils.py strip [-h]
                          input_file_path output_file_path parameters
                          [parameters ...]

    positional arguments:
      input_file_path   Input hdf filename.
      output_file_path  Output hdf filename.
      parameters        Store this list of parameters into the output hdf file.
                        All other parameters will be stripped.

    optional arguments:
      -h, --help        show this help message and exit

Example::

    python HDFAccess/hdfaccess/utils.py strip input.hdf5 output.hdf5 Airspeed "Altitude STD"

*input.hdf5* is the input filename, *output.hdf5* is the output filename and
the parameter names which follow will be copied to the output file - all other
parameters will be stripped. Parameter names which contain spaces must be 
wrapped in double quotes.

The utility will not raise an error if specified parameters are missing from
the input file, but will instead output which parameters are in the output
file::

    The following parameters are in the output hdf file:
     * Airspeed
     * Altitude STD

If no parameters are successfully copied then the following message will be 
displayed::

    No matching parameters were found in the hdf file.

Please check that the output hdf file is an acceptable size before adding it
to the repository and that it has been de-identified.

----------------------------------------------
Writing Test Cases Which Call process_flight()
----------------------------------------------

*process_flight* will store DerivedParameterNodes into the HDF file being
processed. If we consider the following test

.. code-block:: python
    :linenos:

    def test_process_flight(self):
        process_flight('test_data.hdf5', aircraft_info)
        with hdf_file('test_data.hdf5') as hdf:
            self.assertEqual(hdf['Altitude AAL'].array, expected_result)

When this test is run for the first time, process_flight will create 
*Altitude AAL* in *test_data.hdf5*. The second time the test is run,
*Altitude AAL* will already exist within the file and therefore will not be
processed. Changes to the AnalysisEngine codebase will no longer affect the
result of this test.

To avoid this problem, we should first copy the file and run process_flight
against the copy

.. code-block:: python
    :linenos:

    from flightdatautilities.filesystem_tools import copy_file
    
    ...
    
    def test_process_flight(self):
        hdf_copy = copy_file('test_data.hdf5', postfix='_test_copy')
        process_flight(hdf_copy, aircraft_info)
        with hdf_file(hdf_copy) as hdf:
            self.assertEqual(hdf['Altitude AAL'].array, expected_result)

In this case *Altitude AAL* will be processed each time the test is run.


Usefull Functions
=================


tests.derived_parameter_tests.assert_array_within_tolerance
    
    Check that the actual array within tolerance of the desired array is
    at least similarity percent.

:py:func:`analysis_engine.node.Node.get_operational_combinations`

    .. code-block:: python

    from analysis_engine.derived_parameters import AltitudeAAL
    alt_aal = AltitudeAAL()
    alt_aal.get_operational_combinations()

:py:func:`analysis_engine.node.Node.save`
    Method for pickling and saving a node to file.

    .. code-block:: python

        import numpy as np
        from analysis_engine.node import M
        
        spd_brk = M(name='Speedbrake Selected',
                    array=np.ma.array([0, 1, 2, 0, 0] * 3),
                    values_mapping={
                        0: 'Stowed',
                        1: 'Armed/Cmd Dn',
                        2: 'Deployed/Cmd Up',
                    },)
        spd_brk.save('/home/parke1r/spd_brk.nod')

:py:func:`analysis_engine.node.load`
    Loads a prevously saved pickled node from a file.

    .. code-block:: python

        from analysis_engine.node import load
        spd_brk2 = load('/home/parke1r/spd_brk.nod')
