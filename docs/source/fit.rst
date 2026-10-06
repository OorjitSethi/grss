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
