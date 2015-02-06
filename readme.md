Building ITN for pgRouting
==========================

This document and repository explains how to load Ordnance Survey ITN GML into a PostGIS database with Loader and then build a network suitable for use with pgRouting.  The network will have one way streets, no entry and no turn restrictions, mandatory turns and grade separations for over- and underpasses.  Network costs will be calulated using average travel speeds.  Further realism can be added to the network using more of the road routing information supplied with ITN.  I'll leave that to better brains than mine.

Setting up PostGIS
------------------
I am running this on PostgreSQL 9.3.5 with PostGIS 2.1.5 and pgRouting 2.0.0 on "localhost" with user "postgres".  Adjust your settings as necessary.  This is what I used.

Create a new database: **routing**

Create a new schema: **osmm_itn**

Adjust the search_path variable for user postgres if required to access the osmm_itn schema.

Enable the pgRouting extension: **CREATE EXTENSION pgrouting;**

Configuring Loader
------------------
Get Loader: https://github.com/AstunTechnology/Loader and download and unpack into your working directory.  Follow the instructions to configure Loader to load the ITN GML into PostGIS.

My Loader configuration (sans comments)

    src_dir=C:\Workspace\Loader\ITN_6348787\01
    out_dir=C:\Workspace\Loader\output
    tmp_dir=C:\Workspace\Loader
    ogr_cmd=ogr2ogr --config GML_EXPOSE_FID NO -append -skipfailures -f PostgreSQL PG:'dbname=routing active_schema=osmm_itn host=localhost user=postgres password=yourpassword port=5432' $file_path
    prep_cmd=python prepgml4ogr.py $file_path prep_osgml.prep_osmm_itn
    post_cmd=
    gfs_file=../gfs/osmm_itn_postgres.gfs
    debug=False

Loading
-------
Load the ITN GML into PostGIS using Loader by running in the Loader directory:

    python loader.py loader.config

Note: I used non-geographically chunked GML (as opposed to geographically chunked) from Ordnance Survey - this does not contain duplicate features which can lead to issues later on.

Once the GML has been loaded into PostGIS run the views.sql file in the extras directory of the Loader installation.  This creates a number of views that link the ITN tables together and provide the links between the road links and the road route information (RRI).

Now you should have some tables and some views in the osmm_itn schema in your routing database.

To build a valid network that pgRouting can use you need to create some additional views which contain the information required to model one way streets, grade separations and turn restrictions and mandatory turns.

Update road names and road numbers
----------------------------------
First let's update the roadlink table with road names and road numbers.  Add the roadname and roadnumber fields to the roadlink table:

    ALTER TABLE roadlink ADD COLUMN roadname character varying(250);
    ALTER TABLE roadlink ADD COLUMN roadnumber character varying(70);

This uses the road table to update the roadlink table with road names.  Takes some time to run if you have a large dataset.

    UPDATE roadlink SET roadname = 
    (
    SELECT DISTINCT array_to_string(c.roadname,', ') AS roadname 
    FROM roadlink a 
    INNER JOIN road_roadlink b ON b.roadlink_fid = a.fid
    INNER JOIN road c ON c.fid = b.road_fid
    WHERE a.fid = roadlink.fid 
    AND c.descriptivegroup = 'Named Road'
    );

The following three queries update the roadlink table with the DFT road numbers.  This is somewhat quicker than the query above as there usually fewer records to update.

Update Motorways

    UPDATE roadlink SET dftname = 
    (
    SELECT DISTINCT array_to_string(c.roadname,', ') AS roadname 
    FROM roadlink a 
    INNER JOIN road_roadlink b ON b.roadlink_fid = a.fid
    INNER JOIN road c ON c.fid = b.road_fid
    WHERE a.fid = roadlink.fid 
    AND c.descriptivegroup = 'Motorway'
    )
    WHERE roadlink.descriptiveterm = 'Motorway';

Update A Roads

    UPDATE roadlink SET dftname = 
    (
    SELECT DISTINCT array_to_string(c.roadname,', ') AS roadname 
    FROM roadlink a 
    INNER JOIN road_roadlink b ON b.roadlink_fid = a.fid
    INNER JOIN road c ON c.fid = b.road_fid
    WHERE a.fid = roadlink.fid 
    AND c.descriptivegroup = 'A Road'
    )
    WHERE roadlink.descriptiveterm = 'A Road';

Update B Roads

    UPDATE roadlink SET dftname = 
    (
    SELECT DISTINCT array_to_string(c.roadname,', ') AS roadname 
    FROM roadlink a 
    INNER JOIN road_roadlink b ON b.roadlink_fid = a.fid
    INNER JOIN road c ON c.fid = b.road_fid
    WHERE a.fid = roadlink.fid 
    AND c.descriptivegroup = 'B Road'
    )
    WHERE roadlink.descriptiveterm = 'B Road';

Create one way view
-------------------
![One Way](images/652.jpg)
Start with creating a view of one way streets in the network.  This combines the information in the roadrouteinformation table with the view linking the roadlinks and roadrouteinformation to select out the streets with a "one way" environmental qualifier.  It also adds a numeric value to the roadlink to indicate whether the one way direction is the same as the digitised direction of the link as shown by the "+" and "-".  These values will be used later in the network table.

    -- View: view_itn_oneway
    -- DROP VIEW view_itn_oneway;
    CREATE OR REPLACE VIEW view_itn_oneway AS 
    SELECT replace(array_to_string(rri.directedlink_href, ', '::text), '#'::text, ''::text) AS directedlink_href,
      rrirl.roadrouteinformation_fid,
      array_to_string(rri.directedlink_orientation, ', '::text) AS directedlink_orientation,
      array_to_string(rri.environmentqualifier_instruction, ', '::text) AS environmentqualifier,
        CASE
            WHEN rri.directedlink_orientation::text = '{+}'::text THEN 512
            ELSE 1024
        END AS oneway_attr
    FROM roadrouteinformation_roadlink rrirl
    RIGHT JOIN roadrouteinformation rri ON rri.fid::text = rrirl.roadrouteinformation_fid::text
    WHERE rri.environmentqualifier_instruction = '{"One Way"}'::character varying[];
    ALTER TABLE view_itn_oneway
      OWNER TO postgres;
    COMMENT ON VIEW view_itn_oneway 
      IS 'ITN one way streets view';

Create grade separation view
----------------------------
<img src="https://github.com/mixedbredie/itn-for-pgrouting/raw/master/images/530A.JPG" alt="Grade Separation" width="206px">
Then create a view to hold all the links with grade separation values of 1, i.e. elevated at one or both ends.  The view will be used to identify all the links in the final network table that make up bridges and overpasses.

    -- View: view_itn_gradeseparation
    -- DROP VIEW view_itn_gradeseparation;
    CREATE OR REPLACE VIEW view_itn_gradeseparation AS 
      SELECT rl.fid,
      rl.ogc_fid,
      rl.directednode_gradeseparation[1] AS gradeseparation_s,
      rl.directednode_gradeseparation[2] AS gradeseparation_e,
      COALESCE(((rl.roadname::text || ' ('::text) || rl.dftname::text) || ')'::text, COALESCE(rl.dftname::text, rl.descriptiveterm::text)) AS roadname,
      rl.wkb_geometry
    FROM roadlink rl
    WHERE rl.directednode_gradeseparation = '{0,1}'::integer[] OR rl.directednode_gradeseparation = '{1,1}'::integer[] OR rl.directednode_gradeseparation = '{1,0}'::integer[];
    ALTER TABLE view_itn_gradeseparation
      OWNER TO postgres;
    COMMENT ON VIEW view_itn_gradeseparation
      IS 'ITN links with grade separation';

This is a work in progress here but pgRouting has some issues with bridges being split where they cross the underlying road link.  So, in an attempt to build a better network the following view takes the road links with grade separation values of 1 and unions then merges the input to create single linestrings representing the bridge.  These are then put back into the final network table replacing the existing road links.

    -- View: view_itn_bridges

    -- DROP VIEW view_itn_bridges;

    CREATE OR REPLACE VIEW view_itn_bridges AS 
     SELECT gs.roadname,
        (st_dump(st_linemerge(st_union(gs.wkb_geometry)))).geom AS wkb_geometry
     FROM view_itn_gradeseparation gs
     GROUP BY gs.roadname;

    ALTER TABLE view_itn_bridges
      OWNER TO postgres;

Create network table
--------------------
Once we have this in place we can create an empty table to hold the route network geometry and attributes required for pgRouting.  I am going to call my table "itn_network".

Create network build function
-----------------------------
Now we will create a function to use the roadlink table and the grade separation and one way views to populate the network table we created in the previous step.

Update one way field
--------------------

Update grade separation fields
------------------------------

Rebuild elevated sections
-------------------------

Populate pgRouting fields
-------------------------

Calculate network costs
-----------------------

Create no entry restrictions
----------------------------
<img src="https://github.com/mixedbredie/itn-for-pgrouting/raw/master/images/616.jpg" alt="No Entry" width="206px">

Create no turn restrictions
---------------------------
<img src="https://github.com/mixedbredie/itn-for-pgrouting/raw/master/images/613.jpg" alt="No Turn" width="206px">

Create mandatory turn restrictions
----------------------------------
![Mandatory Turn](images/609A.jpg)

Build pgRouting topology
------------------------
    
    SELECT pgr_createTopology('osmm_itn.itn_network', 0.001, 'wkb_geometry', 'gid', 'source', 'target');
    
Analyse network topology
------------------------

    SELECT pgr_analyzeGraph('osmm_itn.itn_network', 0.001, 'wkb_geometry', 'gid', 'source', 'target'); 

pgRouting and QGIS
------------------

References
----------
http://www.ordnancesurvey.co.uk/business-and-government/products/itn-layer.html

https://github.com/AstunTechnology/Loader

http://pgrouting.org/

http://www.archaeogeek.com/blog/2012/08/17/pgrouting-with-ordnance-survey-itn-data/

http://anitagraser.com/?s=pgrouting
