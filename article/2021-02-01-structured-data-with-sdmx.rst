Structuring data with SDMX
**************************

:tags: data, integrated-assessment, python, research, sdmx, software

This post expands on some text from the README of `iam-units <https://github.com/IAMconsortium/units/#readme>`_, in order to illustrate some improved ways to structure data and metadata.
These are especially helpful in systems research, where data from multiple disciplines/domains/contexts are often combined.

``iam-units`` is a thin wrapper around the very useful `pint <https://pint.readthedocs.io>`_, that I wrote with some colleagues to handle unit conversions for data from integrated assessment models (IAMs) of energy and climate, including `global warming potential`_ (GWP) conversions between greenhouse gas (GHG) species:

.. ipython:: python

   from iam_units import registry, convert_gwp

   # Use a pint.UnitRegistry to convert from one unit to another
   qty = registry('3500 t').to('Mt')

   # The result has the expected magnitude: 3500 t = 3.5 kt = 0.035 Mt
   qty

   # Convert from mass of N₂O to GWP-equivalent mass of CO₂,
   # using the metric from the IPCC Fifth Assessment Report (AR5)
   convert_gwp('AR5GWP100', qty, 'N2O', 'CO2')

**Contents**

.. contents::
   :local:
   :backlinks: none

Here's the text I'd like to expand on, with emphasis added:

    ``convert_gwp()`` converts from from mass (or mass-related units) of one specific greenhouse gas (GHG) species to an equivalent quantity of second species, based on GWP *metrics*.

    The function also accepts input with the commonly-used combination of mass (or related) units and the identity of a particular GHG species:

    Strictly, **the original species is not a unit but a “nominal property”**; see the `International Vocabulary of Metrology`_ (VIM) [1]_ used in the `SI`_.
    To avoid ambiguity, code handling GHG quantities should also track and output these nominal properties, including:

    1. Original species.
    2. Species in which GWP-equivalents are expressed (e.g. CO₂ or C)
    3. GWP metric used to convert (1) to (2).

.. _global warming potential: https://en.wikipedia.org/wiki/Global_warming_potential
.. _International Vocabulary of Metrology: https://www.bipm.org/utils/common/documents/jcgm/JCGM_200_2008.pdf
.. _SI: https://en.wikipedia.org/wiki/International_System_of_Units

This is one of many cases where a little up-front effort to be precise—as opposed to loose and casual—about measurement can help us write code and produce data that is as interoperable as possible.
The investment pays dividends by reducing repeated work down the road in handling imprecise data.

Concept, observation, quantity, magnitude, unit
===============================================

Before returning to GHGs, here's a second example: in transport research we often want to discuss the following *concept*:

    total distance traveled by a person (or group of people)

For instance, if I walk to the corner store, ride my bike to a restaurant, and then take the train to visit a friend, I might travel in total 0.5 + 4 + 10 = 14.5 kilometres.
This is a *quantity* that expresses one *observation* of the above concept.
The observation was, hypothetically, obtained by a process of *measurement* (more on that shortly).
The quantity consists of the *magnitude* “14.5” and *unit reference* “kilometre”.

We might also say:

.. ipython:: python

   registry("14.5 km").to("mile")

This is the very same *concept*, *observation*, and *quantity*, but expressed in a different *unit* and thus having a different *magnitude*. [2]_

Measurement and nominal properties
----------------------------------

Something is still missing in order to be unambiguous about the observation “the travel distance is 14.5 kilometres.”
We might ask:

- Is this the actual distance traveled in a specific period of time?

  - What period? A specific day? [3]_
  - Or, as in this case, a hypothetical, counterfactual, etc.?

- Is it a computed statistic? For instance:

  - The *mean* or *average* travel distance across 2+ periods, for just one person (me)?
  - The *sum*/collective travel distance across 2+ people, but in the same period?
  - Something else?

The VIM calls these **nominal properties**.

In systems research, it is common to combine data and knowledge from different streams of research that take place in different contexts, with different scope and resolution.
*Within* any particular context, there can be certain natural choices for concepts, measurement process(es), and thus certain nominal properties of the data created/handled.
These will be understood implicitly by both author and audience as the background consensus, and don't vary within that context.
To list them explicitly doesn't achieve much, and might even be a distraction and barrier to clear communication; noise that buries the signal.

However: when combining data from different contexts, problems can arise when we fail to notice that the concepts, measurement processes, and nominal properties differ.
A burden thus falls on the researcher to:

1. Make explicit the implicit features of measurement in the different domains supplying data that she wishes to combine,
2. Identify any differences,
3. Choose methods to adjust for (2), and
4. Correctly implement (3).

These tasks consume resources and create opportunities for errors that could undermine the internal validity of research.
But they're also routine, so we can use some judicious automation to reduce the work involved while getting equally- or more-valid results.

Common ways to handle GHGs and GWP-equivalents
==============================================

Suppose we have this data:

.. ipython:: python

    import pandas as pd

    species = ["N2O", "CH4", "CO2"]

    data_a = pd.DataFrame(
        [
            ["Emissions, N2O", 1.1, "kt"],
            ["Emissions, CH4", 5.2, "kt"],
            ["Emissions, CO2", 100.3, "kt"],
        ],
        columns=["variable", "value", "unit"],
    )

    data_b = pd.DataFrame(
        [
            ["Emissions, N2O", 291.5, "kt"],
            ["Emissions, CH4", 146.5, "kt"],
            ["Emissions, CO2", 100.3, "kt"],
        ],
        columns=["variable", "value", "unit"],
    )

Let's be very precise about what we have:

- Each observation in ``data_a`` expresses a measured mass of emissions.
- Each observation in ``data_b`` expresses a mass of CO₂ that, using the AR5 100-year metric, has the potential to contribute the same amount of global warming as the corresponding mass in ``data_a``.
- The ‘value’ column contains *magnitudes*; actual *quantities* are given by the ‘value’ and ‘unit’ columns together.
- The ‘variable’ column mixes concepts: the thing measured is the [mass] emitted and is the same for all observations; the species emitted varies.

Clearly, it's wrong to do the following:

.. ipython:: python

    data_a["value"] + data_b["value"]

These magnitudes, as bare numbers, can of course be added togther.
But the results are scientifically meaningless, since the operands are conceptually incoherent: their units are the same, but they measure different things.

There are some common ways to store information that is intended to help avoid errors like this.
Strategy A is to use a *unit-like expression* that combines the actual *unit* with a nominal property, i.e. the species:

.. ipython:: python

    data_a.assign(unit=[f"kt {s}" for s in species])

    data_b.assign(unit="kt CO2-eq")

These expressions are a common prose shorthand or abbreviation, but are not standard.
For instance, ‘-e’/‘e’, ‘-eq’/‘eq’, ‘-equiv’, and other symbols are all in use in various contexts.

Strategy B is to mash even more concepts into a column named something like ‘variable’, adding (3) and (4) to (1) and (2) here:

1. The property that is measured, i.e. mass.
2. The species emitted.
3. The species in which a GWP-equivalent is expressed.
4. The GWP metric used.

.. ipython:: python

    data_a

    data_b.assign(
        variable=data_b.variable + " (CO₂ equivalent, AR5GWP100)"
    )

Both of these strategies have the same basic flaw: they *combine* instead of *distinguishing* different aspects of measurement.
Researchers might make choices that feel ‘natural’ in specific contexts, yet which differ from equally natural choices made by others.
This creates a proliferation of idiosyncratic formats, and entails further work in parsing and harmonizing.

Using SDMX to be explicit
=========================

I maintain `sdmx1 <https://sdmx1.readthedocs.io>`_, a Python package that implements the SDMX Information Model, or ISO 17369:2013.
An “information model” (IM) is a *model* (a set of concepts and their relationships) for talking about *information* (data and metadata) and its representation.
(The documentation for the package links to some `learning resources for SDMX <https://sdmx1.readthedocs.io/en/latest/resources.html>`_; the details aren't repeated here.)

Because it is carefully developed, the SDMX IM allows us to precisely capture the concepts and relationships in our example data, including our choices in measurement and of units.

The rest of this post gives a minimal demonstration.

Specify the concepts and data structure
---------------------------------------

We start by defining each of the distinct concepts that occur in our data:

.. ipython:: python

    import sdmx
    from sdmx import model

    emission = model.Concept(
        id="EMISSION",
        name="Mass of greenhouse gas emitted",
    )
    species_concept = model.Concept(
        id="SPECIES",
        name="Chemical species or substance",
    )
    gwp_metric = model.Concept(
        id="GWP_METRIC",
        name="Set of GWPs used to convert species",
    )
    # “UNIT_MEASURE” is commonly used in SDMX applications
    unit_concept = model.Concept(
        id="UNIT_MEASURE",
        name="Unit of measurement",
    )


Next, we define the structure of our data.
We start with the *primary measure*, the concept that is measured:

.. ipython:: python

    dsd = model.DataStructureDefinition(id="GHG_DATA")

    # - Refer to the concept
    # - Use concept's ID for the primary measure as well
    pm = model.PrimaryMeasure(
        concept_identity=emission,
        id=emission.id,
    )

    # Store in the DSD
    dsd.measures.append(pm)

Next, we specify that values for the ``gwp_metric`` and ``unit`` concepts are stored as *attributes*.
SDMX allows several options for where we attach these attributes.
Here, we specify that the attributes are attached to entire data sets.
This means that, in a particular data set, only one GWP metric can be used, and one unit: [4]_

.. ipython:: python

    # 1. Define the data attribute
    #    - Refer to the concept
    #    - Use concept's ID for the attribute as well
    #    - NoSpecifiedRelationship = attached to data set
    # 2. Store in the DSD
    for concept in (gwp_metric, unit_concept):
        da = model.DataAttribute(
            concept_identity=concept,
            id=concept.id,
            related_to=model.NoSpecifiedRelationship(),
        )
        dsd.attributes.append(da)

Finally, we specify that our data has two dimensions.
Every observation in a data set must have a unique Key with values for these dimensions.

Both dimensions refer to the same concept (species), but we give them different IDs and names to explain what these signify.

.. ipython:: python

    for order, (id, name) in enumerate((
        ("SPECIES", "Original species emitted"),
        (
            "SPECIES_GWP",
            "Species in which GWP equivalent is expressed",
        ),
    )):
        dim = model.Dimension(
            concept_identity=species_concept,
            id=id,
            name=name,
            order=order,
        )
        dsd.dimensions.append(dim)

We now have a complete description of our data structure:

.. ipython:: python

    dsd.measures
    dsd.attributes
    dsd.dimensions

Use the definitions to structure the data
-----------------------------------------

Now we can create a data set that is *structured by* this definition:

.. ipython:: python

    ds_a = model.DataSet(structured_by=dsd)

We store the values of attributes attached to the data set:

.. ipython::

    # Define a function to create attribute values and store them.
    # The `value_for` property links each value to the definition
    # of the attribute in the DSD, and thus to a concept.
    In [1]: def store_attributes(ds, **values):
       ...:     for concept_id, value in values.items():
       ...:         av = model.AttributeValue(
       ...:             value=value,
       ...:             value_for=dsd.attributes.get(concept_id),
       ...:         )
       ...:         ds.attrib[concept_id] = av

    In [2]: store_attributes(
       ...:     ds_a,
       ...:     GWP_METRIC="(None)",
       ...:     UNIT_MEASURE="kt",
       ...: )

Finally we store the individual observations.
In the case of ``data_a``, the labels for the SPECIES and SPECIES_GWP dimensions are the same:

.. ipython:: python

    # 1. Discard redundant portion of the ‘variable’ column.
    # 2. Create a key object from labels for each dimension.
    # 3. Create the observation.
    #    - The value is ‘for’ the primary measure, i.e. emissions.
    # 4. Store in the data set.
    for _, row in data_a.iterrows():
        s = row["variable"].split("Emissions, ")[-1]
        dims = dict(SPECIES=s, SPECIES_GWP=s)
        key = dsd.make_key(model.Key, dims)
        obs = model.Observation(
            dimension=key, value=row["value"], value_for=pm
        )
        ds_a.obs.append(obs)

We now have a complete data set.
``sdmx1`` provides features for converting to other data structures and file formats:

.. ipython:: python

    ds_a

    # pandas.DataFrame
    sdmx.to_pandas(ds_a, attributes="od")

    # Standardized SDMX-ML format
    sdmx.to_xml(ds_a, pretty_print=True)

Convert carefully
-----------------

Finally, we can convert ``ds_a`` using ``iam-units`` while maintaining an explicit and unambiguous data structure.

We create a second data set using the same data structure definition:

.. ipython:: python

    ds_b = model.DataSet(structured_by=dsd)

    # Store attributes:
    # GWP metric name to use in conversion
    metric = "AR5GWP100"

    # Unit used in both data sets
    unit = ds_a.attrib["UNIT_MEASURE"].value

    store_attributes(
        ds_b, GWP_METRIC=metric, UNIT_MEASURE=unit
    )

For each observation, we convert its magnitude, and change *only* the “SPECIES_GWP” key value:

.. ipython:: python

    # Target species for conversion
    target_species = "CO2"

    # 1. Store the original species.
    # 2. Create a new key with SPECIES_GWP set to `target_species`.
    # 3. Convert the magnitude using the metric and iam-units.
    # 4. Create a new observation and store in `ds_b`.
    for obs in ds_a.obs:
        species = obs.dimension["SPECIES"].value
        key = dsd.make_key(
            model.Key,
            dict(SPECIES=species, SPECIES_GWP=target_species),
        )
        value = convert_gwp(
            metric,
            registry.Quantity(obs.value, unit),
            species,
            target_species,
        ).magnitude
        ds_b.obs.append(
            model.Observation(
                dimension=key, value=value, value_for=pm
            )
        )

The resulting data set has the same format as ``ds_a``:

.. ipython:: python

    ds_b

    sdmx.to_pandas(ds_b, attributes="do")

Take-aways
==========

To review, by applying the SDMX Information Model, we were able to:

1. Define exactly the concepts relevant to our data.
2. Use these to create a clear and unambigous structure, in which concepts were used for the primary measure, as attributes, or dimensions of multi-dimensional data.
3. Unpack a ‘variable’ column into its constituent concepts, and avoid overloading it further with additional concepts.
4. Store simple or bare SI units, parseable by ``pint``, for the “UNIT_MEASURE” attribute, and avoid overloading this with unrelated concepts.
5. Access structured metadata and use it in the process of converting data.

Further topics
--------------

The amount of code used to handle only 3 observations might seem excessive.
The up-front investment, however, unlocks further improvements in handling data and metadata.
As a sketch of these:

- Applications can additionally specify a *representation* for any concept used for a measure, attribute, or dimension.

  For instance, we could specify that the “SPECIES” dimension represents the concept of species using only codes from a certain list of species understood to be GHGs: CH4, CO2, N2O, and so on.
  The code list is a mechanism for communicating about the data:

  - “This data contains codes that are invalid, not in the specified list” can flag errors in data preparation.
  - A data provider can specify a subset of the codes that will appear in their published data sets or data flows.
    Users then understand not to expect values outside this subset.

- The concepts and other structures can be shared and reused.
  For instance, as described in footnote [4]_, we could construct a *different* data structure definition using the *same* concepts, as required for a different application.

  The re-use of the same concepts tells users about links between multiple data sets and flows.

- Published structures allow multiple data providers to prepare data that is guaranteed to be interoperable.

----

**Footnotes**

.. [1] The VIM is good reading and food for thought on these topics; more giant shoulders that we stand on, even if it seems as pedestrian as a dictionary.
   I encourage everyone to skim it!

.. [2] It's very common to conflate concepts and units.
   Do not do this.

   For instance, “passenger/person distance traveled” (PDT) is often called “PKT” (meaning “passenger kilometres travelled”) or “PMT” (“person-miles traveled”).
   This conflates units (kilometres, miles) with the quantity (distance) measured.
   Two data sets with observations labelled “PKT” and “PMT” might measure the same concept—that is, PDT—in the same way with the same nominal properties, but merely express the resulting quantities in different units.

   Always label dimensions or observations with the *concept measured*, not the *units* of particular measurements.

.. [3] …or, during this pandemic, month or year? 😭️

.. [4] This is simply for brevity, not a limitation of SDMX.

   We could equally, for instance, specify that ``gwp_metric`` is attached to series keys or individual observations.
   This would allow a single data set to contain the same measurements with magnitudes expressed in multiple ways via multiple GWP metrics.
