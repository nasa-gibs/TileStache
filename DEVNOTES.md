Tilestache / Mapnik installation instructions to render Mapnik-styled MVTs to image tiles on OS X
- uninstalled python in /Library/Frameworks (note that’s different than /System/Library/Frameworks) and ensured brew’s python was primary

- installed custom boost and boost-python libraries to use [this](https://github.com/mapnik/mapnik-packaging/commit/4ded033e29a05c929fa8c5d770a1dcc3704f9113) patch ([more official here?](https://github.com/mapnik/mapnik-packaging/blob/master/osx/patches/boost_python_shared_ptr_gil.diff)) as referenced in these ([1](https://github.com/mapnik/mapnik/issues/2056), [2](https://github.com/mapnik/mapnik/issues/1968)) issues;  due to many linking etc problems, did this through modified homebrew install;  basically repackaged homebrew’s install packages to include the patch

- installed semi-custom mapnik2 library through homebrew;  default installation didn’t build/install python datasource plugin and couldn’t force INPUT_PLUGINS=‘all’ during the configure step;  so inserted a pause during installation after configuration step (called python from command line) and re-ran configuration in the homebrew working directory (/private/tmp/…) with:

      python scons/scons.py configure CC="clang" CXX="clang++" JOBS=4 PREFIX=/usr/local/Cellar/mapnik2/2.2.0_2 ICU_INCLUDES=/usr/local/opt/icu4c/include ICU_LIBS=/usr/local/opt/icu4c/lib PYTHON_PREFIX=/usr/local/Cellar/mapnik2/2.2.0_2 JPEG_INCLUDES=/usr/local/opt/jpeg/include JPEG_LIBS=/usr/local/opt/jpeg/lib PNG_INCLUDES=/usr/local/opt/libpng/include PNG_LIBS=/usr/local/opt/libpng/lib TIFF_INCLUDES=/usr/local/opt/libtiff/include TIFF_LIBS=/usr/local/opt/libtiff/lib BOOST_INCLUDES=/usr/local/opt/boost159/include BOOST_LIBS=/usr/local/opt/boost159/lib PROJ_INCLUDES=/usr/local/opt/proj/include PROJ_LIBS=/usr/local/opt/proj/lib FREETYPE_CONFIG=/usr/local/opt/freetype/bin/freetype-config CAIRO=False INPUT_PLUGINS='all'

- configured environment and install libraries

      # ensure mapnik2 is picked up in homebrew's python path
      export PYTHONPATH="/usr/local/lib/python2.7/site-packages"

      # force use of OSX version of python (rather than homebrew) since something
      # apparently depends on it (otherwise causes a segfault at nik2img runtime)
      virtualenv --python=/System/Library/Frameworks/Python.framework/Versions/2.7/bin/python --system-site-packages env
      source env/bin/activate

      # Install modified Tilestache which can read protobuf-enabled MVTs
      git clone https://github.com/nasa-gibs/TileStache.git
      cd TileStache
      python setup.py install

      # Install mapbox vector tile module
      pip install mapbox-vector-tile

      # Install nik2img ( https://github.com/springmeyer/nik2img )
      pip install nik2img

      # Install vt-geojson ( https://github.com/developmentseed/vt-geojson );  global install necessary due to its use on the command line
          npm install -g vt-geojson


- Running

      export MapboxAccessToken=<insert your token here>
      nik2img.py mapbox-tilejson-simple.xml nik2img-output991.png -v -z 2 -c -40 70 -d 800 800


CURRENT DEV STEPS
- create script to convert Mapbox Studio Classic / Mapnik XML files to TileStache-ready format; basically add the following before the `</Layer>` directive:

          <Datasource>
            <Parameter name="type">python</Parameter>
            <Parameter name="factory">TileStache.Goodies.VecTiles:Datasource</Parameter>
            <Parameter name="template">http://a.tiles.mapbox.com/v4/mapbox.mapbox-streets-v7/{z}/{x}/{y}.mvt?access_token=<insert your token here></Parameter>
            <Parameter name="sort_key">sort_key</Parameter>
          </Datasource>

- may also want to not require access token to be in Mapnik XML files and instead try to pull from $MapboxAccessToken environment variable.  tried to “$MapboxAccessToken” directly within XML file but it just got picked up as a literal

- create script to programmatically generate tiles at GIBS-compatible tile grids


- ALTERNATIVELY: use [mapnik-vector-tile](https://github.com/mapbox/mapnik-vector-tile) to read mapnik stylesheets and render images;  probably should do that anyway to fix possible styling issues based on old version of mapnik and horrible installation processes to get there.  It requires mapnik3 built from master, though, as opposed to homebrew’s version

CURRENT EVALUATION STEPS
- bring in a more complex mapnik stylesheet example from mapbox studio classic and see how it renders; may also want to look into nik2img to validate what it's doing 
- if the above styling doesn’t work well with mapnik 2.x, use mapnik3 and add-on vector tile library;  write mapbox for advice on using it + mapnik to render xml stylesheets to images;  MAYBE EVEN USE [tilelive-mapnik](https://github.com/mapbox/tilelive-mapnik)?  probably only supports web merc.   also there’s a nice diagram of the tilelive ecosystem [here](https://github.com/mapbox/tilelive#ecosytem-of-tilelive).


Misc notes
- may need to install xcode command line tools to install Pillow (xcode-select --install)

- now detecting mvt requests in tilestache (TileStache/Goodies/VecTiles/client.py in load_tile_features function), converting those requests to vt-geojson ones, then forcing tilestache to read them

- example of converting pbf mvts using [vt-geojson](https://github.com/developmentseed/vt-geojson) (formerly got "Error: Unimplemented type: 3” from the pbf interpreter but works now) (need to set mapbox access key token):

      export MapboxAccessToken=<insert your token here>
      vt-geojson mapbox.mapbox-streets-v7 --tile 4823 6160 14 > testout.geojson
