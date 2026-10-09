GRSS Orbit Determination Module (grss.fit)
==========================================
The orbit determination code within GRSS is completely on the Python side of things, but it heavily relies on the C++ propagator binding. Given a small body orbit that needs to be fitted to a set of observations, the module uses the batch least squares algorithm to solve for an updated nominal orbit.

The observation preparation and fitting subfunctions are available through the
Python ``grss.fit`` package. These functions run in the caller's Python process;
they do not require an HTTP server. For example::

    from grss.fit import apply_weighting_scheme

    weighted_obs = apply_weighting_scheme(obs_df, verbose=False)

``obs_df`` must be a GRSS optical observation DataFrame with the columns
described in the function's API documentation. The weighting function updates
that DataFrame in place and returns it. Other preparation functions have their
own input requirements and may also mutate the objects passed to them; read
their individual API entries before calling them outside ``get_optical_obs``.
Underscore-prefixed helpers remain internal implementation details.

The public fitting workflow is composed of these callable stages:

* **Acquire observations:** ``get_mpc_raw_data``, ``get_radar_raw_data``,
  and ``get_gaia_query_results`` retrieve source data.
* **Validate and assemble:** ``validate_ades_mode`` and
  ``flag_unrecognized_ades_catalogs`` inspect ADES fields;
  ``create_optical_obs_df``, ``add_psv_obs``, ``add_gaia_obs``, and
  ``add_radar_obs`` build the observation table.
* **Correct observations:** ``debias_obs``, ``apply_debiasing_scheme``,
  ``apply_station_weight_rules``, ``apply_weighting_scheme``, ``deweight_obs``,
  and ``eliminate_obs`` handle biases and measurement weights.
* **Fit an orbit:** ``get_optical_obs`` prepares the usual optical workflow,
  and ``FitSimulation`` performs the iterative fit. Conversion and observer
  utilities are also exported from ``grss.fit``.

``flatten_valid_observations`` is an independent array helper used when
calculating numerical partial derivatives. It flattens a numeric observation
array and removes ``NaN`` entries without changing the input::

    from grss.fit import flatten_valid_observations

    valid_values = flatten_valid_observations([[1.0, float('nan')], [2.0, 3.0]])

An initialized ``FitSimulation`` also exposes three solution-conversion
methods: ``solution_to_state(x_dict)``,
``solution_to_nongrav_params(x_dict)``, and ``solution_to_events(x_dict)``.
These return the state vector, non-gravitational parameter object, and event
tuples used by the propagator. They do not change the fitter's state; the
fitter's Cartesian/cometary mode and fixed propagation parameters determine
how a supplied solution dictionary is interpreted.

For numerical derivatives, ``get_perturbed_state(key)`` returns the positive
and negative solutions for one fitted parameter plus the finite-difference
step. ``get_perturbation_info()`` returns those tuples for every parameter in
the current nominal solution, in solution-key order. Neither method changes
the nominal solution.

For an initialized ``FitSimulation``, ``compute_residuals_and_partials()``
evaluates the current nominal orbit without applying a least-squares state
correction. It returns the residuals and partial derivatives, stores the
propagated simulations, and updates the observation table's residual columns.
``filter_lsq()`` calls this same method during each fitting iteration.
The main linearized fitting stages can also be called explicitly on an
initialized ``FitSimulation``::

    fit.prepare_priors()
    residuals, partials = fit.compute_residuals_and_partials()
    rms_u, rms_w, chi_sq = fit.compute_fit_statistics(partials, residuals)
    delta_x = fit.solve_state_correction(partials, residuals)

These calls expose the intermediate results. ``solve_state_correction``
returns a proposed correction and does not update the nominal orbit;
``filter_lsq`` manages the complete iteration, including applying corrections,
convergence checks, and outlier rejection. Outlier rejection through
``compute_fit_statistics(..., start_rejecting=True)`` requires a covariance
from a previous correction.

Lower-level ``FitSimulation`` methods are public for callers that need to
inspect or compose individual stages. Their required order and side effects
matter because a fitter stores observations, weights, native simulations, and
iteration history between calls:

* **Setup:** ``check_initial_solution``, ``add_simulated_obs``,
  ``parse_observation_arrays``, and ``compute_obs_weights`` prepare cached fit
  state. The constructor runs initial-solution validation, observation parsing,
  and weight construction. Repeated parsing can append simulated observations
  again; after changing observations, rebuild weights before propagation.
* **Propagation:** ``get_prop_sim_past``, ``get_prop_sim_future``, and
  ``get_prop_sims`` create configured native simulations;
  ``check_and_add_events`` adds events to those objects; and
  ``assemble_and_propagate_bodies`` integrates nominal and perturbed bodies.
  The past or future simulation is ``None`` when no observations fall on that
  side of the solution epoch.
* **Measurements and derivatives:** ``get_computed_obs`` reads propagated
  measurements; ``inflate_uncertainties`` updates optical covariance and
  weights; ``get_analytic_partials`` and ``get_numeric_partials`` calculate
  derivatives; and ``get_partials`` selects the configured derivative method.
  These methods require integrated simulations. Computing nominal observations
  also inflates applicable uncertainties.
* **Iteration history:** ``add_iteration`` stores a fit snapshot and
  ``check_convergence`` updates the convergence flag after a correction.

The old underscore-prefixed spellings remain as compatibility aliases. The
constructor and complete ``filter_lsq`` workflow use the public spellings.

The optical measurements are acquired using the `Minor Planet Center API <https://minorplanetcenter.net/mpcops/documentation/observations-api/>`_, and the radar measurements are acquired using the `JPL Small-body Radar API <https://ssd-api.jpl.nasa.gov/doc/sb_radar.html>`_. The optical astrometry is preprocessed to account for the following :

#. Star catalog biases [#]_
#. Measurement weighting [#]_
#. Gaia astrometry handling [#]_

Once the optical astrometry has been processed, the radar astrometry has been acquired, and the initial orbit is provided by the user, the least squares filter can be run. As of now, the filter can fit the nominal state, the nongravitational acceleration parameters, and any impulsive maneuver events. Currently the partial derivatives in the normal matrix are calculated using 1\ :sup:`st`-order central differences by default, but analytical partial derivatives are also a choice offered to the user. The filter also implements an outlier rejection scheme [#]_ to make sure any spurious measurements do not contaminate the fit.

.. rubric:: References
.. [#] Eggl, S., Farnocchia, D., Chamberlin, A.B., and Chesley, S.R., "Star catalog position and proper motion corrections in asteroid astrometry II: The Gaia era", Icarus, Volume 339, Pages 1-17, 2020. https://doi.org/10.1016/j.icarus.2019.113596.
.. [#] Vereš, P., Farnocchia, D., Chesley, S.R., and Chamberlin, A.B., "Statistical analysis of astrometric errors for the most productive asteroid surveys", Volume 296, Pages 139-149, 2017. https://doi.org/10.1016/j.icarus.2017.05.021.
.. [#] Fuentes-Muñoz, O., Farnocchia, D., Naidu, S.P., and Park, R.S., "Asteroid Orbit Determination Using Gaia FPR: Statistical Analysis", AJ, Volume 167, Pages 290-299, 2024. https://doi.org/10.3847/1538-3881/ad4291.
.. [#] Carpino, M., Milani, A., and Chesley, S.R., "Error statistics of asteroid optical astrometric observations", Icarus, Volume 166, Pages 248-270, 2003. https://doi.org/10.1016/S0019-1035(03)00051-4.
