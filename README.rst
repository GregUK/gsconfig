gsconfig
========

gsconfig is a python library for manipulating a GeoServer instance via the GeoServer RESTConfig API. 

Installing
==========

For users: ``pip install gsconfig`` 

For developers: ``git clone git://github.com/opengeo/gsconfig.git && cd gsconfig && python setup.py develop``
(`virtualenv <http://virtualenv.org/>`_ to taste.)

Getting Help
============
There is a brief manual at http://boundlessgeo.github.io/gsconfig/ .
If you have questions, please ask them on the GeoServer Users mailing list: http://geoserver.org/display/GEOS/Mailing+Lists .
Please use the Github project at http://github.com/boundlessgeo/gsconfig for any bug reports (and pull requests are welcome, but please include tests where possible.)

Sample Layer Creation Code
==========================

.. code-block::

    from geoserver.catalog import Catalog
    cat = Catalog("http://localhost:8080/geoserver/")
    topp = self.cat.get_workspace("topp")
    shapefile_plus_sidecars = shapefile_and_friends("states")
    # shapefile_and_friends should look on the filesystem to find a shapefile
    # and related files based on the base path passed in
    #
    # shapefile_plus_sidecars == {
    #    'shp': 'states.shp',
    #    'shx': 'states.shx',
    #    'prj': 'states.prj',
    #    'dbf': 'states.dbf'
    # }
    
    # 'data' is required (there may be a 'schema' alternative later, for creating empty featuretypes)
    # 'workspace' is optional (GeoServer's default workspace is used by... default)
    # 'name' is required
    ft = self.cat.create_featuretype(name, workspace=topp, data=shapefile_plus_sidecars)

Running Tests
=============

Since the entire purpose of this module is to interact with GeoServer, the test suite is mostly composed of `integration tests <http://en.wikipedia.org/wiki/Integration_testing>`_.  
These tests necessarily rely on a running copy of GeoServer, and expect that this GeoServer instance will be using the default data directory that is included with GeoServer.
This data is also included in the GeoServer source repository as ``/data/release/``.
In addition, it is expected that there will be a postgres database available at ``postgres:password@localhost:5432/db``.
You can test connecting to this database with the ``psql`` command line client by running ``$ psql -d db -Upostgres -h localhost -p 5432`` (you will be prompted interactively for the password.)

Here are the commands that I use to reset before running the gsconfig tests::

   $ cd ~/geoserver/src/web/app/
   $ PGUSER=postgres dropdb db 
   $ PGUSER=postgres createdb db -T template_postgis
   $ git clean -dxff -- ../../../data/release/
   $ git checkout -f
   $ MAVEN_OPTS="-XX:PermSize=128M -Xmx1024M" \
   GEOSERVER_DATA_DIR=../../../data/release \
   mvn jetty:run

At this point, GeoServer will be running foregrounded, but it will take a few seconds to actually begin listening for http requests.
You can stop it with ``CTRL-C`` (but don't do that until you've run the tests!)
You can run the gsconfig tests with the following command::

  $ python setup.py test

Instead of restarting GeoServer after each run to reset the data, the following should allow re-running the tests::

   $ git clean -dxff -- ../../../data/release/
   $ curl -XPOST --user admin:geoserver http://localhost:8080/geoserver/rest/reload

More Examples - Updated for GeoServer 2.4+
==========================================

Loading the GeoServer ``catalog`` using ``gsconfig`` is quite easy. The example below allows you to connect to GeoServer by specifying custom credentials.

.. code-block::

    from geoserver.catalog import Catalog
    cat = Catalog("http://localhost:8080/geoserver/rest/")
    cat.username = "admin"
    cat.password = "****"

The code below allows you to create a FeatureType from a Shapefile

.. code-block::

    geosolutions = cat.get_workspace("geosolutions")
    import geoserver.util
    shapefile_plus_sidecars = geoserver.util.shapefile_and_friends("C:/work/gsconfig/test/data/states")
    # shapefile_and_friends should look on the filesystem to find a shapefile
    # and related files based on the base path passed in
    #
    # shapefile_plus_sidecars == {
    #    'shp': 'states.shp',
    #    'shx': 'states.shx',
    #    'prj': 'states.prj',
    #    'dbf': 'states.dbf'
    # }
    # 'data' is required (there may be a 'schema' alternative later, for creating empty featuretypes)
    # 'workspace' is optional (GeoServer's default workspace is used by... default)
    # 'name' is required
    ft = cat.create_featurestore("test", shapefile_plus_sidecars, geosolutions)

This example shows how to easily update a ``layer`` property. The same approach may be used with every ``catalog`` resource

.. code-block::

    ne_shaded = cat.get_layer("ne_shaded")
    ne_shaded.enabled=True
    cat.save(ne_shaded)
    cat.reload()

Deleting a ``store`` from the ``catalog`` requires to purge all the associated ``layers`` first. This can be done by doing something like this:

.. code-block::

    st = cat.get_store("ne_shaded")
    cat.delete(ne_shaded)
    cat.reload()
    cat.delete(st)
    cat.reload()

There are some functionalities allowing to manage the ``ImageMosaic`` coverages. It is possible to create new ImageMosaics, add granules to them,
and also read the coverages metadata, modify the mosaic ``Dimensions`` and finally query the mosaic ``granules`` and list their properties.

The gsconfig methods map the `REST APIs for ImageMosaic <http://docs.geoserver.org/stable/en/user/rest/examples/curl.html#uploading-and-modifying-a-image-mosaic>`_

In order to create a new ImageMosaic layer, you can prepare a zip file containing the properties files for the mosaic configuration. Refer to the GeoTools ImageMosaic Plugin guide
in order to get details on the mosaic configuration. The package contains an already configured zip file with two granules.
You need to update or remove the ``datastore.properties`` file before creating the mosaic otherwise you will get an exception.

.. code-block::

    from geoserver.catalog import Catalog
    cat = Catalog("http://localhost:8180/geoserver/rest")
    cat.create_imagemosaic("NOAAWW3_NCOMultiGrid_WIND_test", "NOAAWW3_NCOMultiGrid_WIND_test.zip")

By defualt the ``cat.create_imagemosaic`` tries to configure the layer too. If you want to create the store only, you can specify the following parameter

.. code-block::

    cat.create_imagemosaic("NOAAWW3_NCOMultiGrid_WIND_test", "NOAAWW3_NCOMultiGrid_WIND_test.zip", "none")

In order to retrieve from the catalog the ImageMosaic coverage store you can do this

.. code-block::

    store = cat.get_store("NOAAWW3_NCOMultiGrid_WIND_test")

It is possible to add more granules to the mosaic at runtime.
With the following method you can add granules already present on the machine local path.

.. code-block::

    cat.harvest_externalgranule("file://D:/Work/apache-tomcat-6.0.16/instances/data/data/MetOc/NOAAWW3/20131001/WIND/NOAAWW3_NCOMultiGrid__WIND_000_20131001T000000.tif", store)

The method below allows to send granules remotely via POST to the ImageMosaic.
The granules will be uploaded and stored on the ImageMosaic index folder.

.. code-block::

    cat.harvest_uploadgranule("NOAAWW3_NCOMultiGrid__WIND_000_20131002T000000.zip", store)

To delete an ImageMosaic store, you can follow the standard approach, by deleting the layers first.
*ATTENTION*: at this time you need to manually cleanup the data dir from the mosaic granules and, in case you used a DB datastore, you must also drop the mosaic tables.

.. code-block::

    layer = cat.get_layer("NOAAWW3_NCOMultiGrid_WIND_test")
    cat.delete(layer)
    cat.reload()
    cat.delete(store)
    cat.reload()

The method below allows you the load and update the coverage metadata of the ImageMosaic.
You need to do this for every coverage of the ImageMosaic of course.

.. code-block::

    coverage = cat.get_resource_by_url("http://localhost:8180/geoserver/rest/workspaces/natocmre/coveragestores/NOAAWW3_NCOMultiGrid_WIND_test/coverages/NOAAWW3_NCOMultiGrid_WIND_test.xml")
    coverage.supported_formats = ['GEOTIFF']
    cat.save(coverage)

By default the ImageMosaic layer has not the coverage dimensions configured. It is possible using the coverage metadata to update and manage the coverage dimensions.
*ATTENTION*: notice that the ``presentation`` parameters accepts only one among the following values {'LIST', 'DISCRETE_INTERVAL', 'CONTINUOUS_INTERVAL'}

.. code-block::

    from geoserver.support import DimensionInfo
    timeInfo = DimensionInfo("time", "true", "LIST", None, "ISO8601", None)
    coverage.metadata = ({'dirName':'NOAAWW3_NCOMultiGrid_WIND_test_NOAAWW3_NCOMultiGrid_WIND_test', 'time': timeInfo})
    cat.save(coverage)

One the ImageMosaic has been configures, it is possible to read the coverages along with their granule schema and granule info.

.. code-block::

    from geoserver.catalog import Catalog
    cat = Catalog("http://localhost:8180/geoserver/rest")
    store = cat.get_store("NOAAWW3_NCOMultiGrid_WIND_test")
    coverages = cat.mosaic_coverages(store)
    schema = cat.mosaic_coverage_schema(coverages['coverages']['coverage'][0]['name'], store)
    granules = cat.mosaic_granules(coverages['coverages']['coverage'][0]['name'], store)

The granules details can be easily read by doing something like this:

.. code-block::

    granules['crs']['properties']['name']
    granules['features']
    granules['features'][0]['properties']['time']
    granules['features'][0]['properties']['location']
    granules['features'][0]['properties']['run']

When the mosaic grows up and starts having a huge set of granules, you may need to filter the granules query through a CQL filter on the coverage schema attributes.

.. code-block::

    granules = cat.mosaic_granules(coverages['coverages']['coverage'][0]['name'], store, "time >= '2013-10-01T03:00:00.000Z'")
    granules = cat.mosaic_granules(coverages['coverages']['coverage'][0]['name'], store, "time >= '2013-10-01T03:00:00.000Z' AND run = 0")
    granules = cat.mosaic_granules(coverages['coverages']['coverage'][0]['name'], store, "location LIKE '%20131002T000000.tif'")
