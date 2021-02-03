## Spatial Databases - Presentation

You can use the [editor on GitHub](https://github.com/Kropiunig/sdb_project/edit/gh-pages/index.md) to maintain and preview the content for your website in Markdown files.

Whenever you commit to this repository, GitHub Pages will run [Jekyll](https://jekyllrb.com/) to rebuild the pages in your site, from the content in your Markdown files.

### sql queries

queries

~~~~sql
--> How many people reach their neares station in less than 5 minutes?

SELECT SUM(nodes_tiles.einwohner)
FROM moessingen_2po_4pgr_vertices_pgr
JOIN nodes_tiles ON moessingen_2po_4pgr_vertices_pgr.id=nodes_tiles.id
WHERE moessingen_2po_4pgr_vertices_pgr.cost < 0.08333333;
~~~~

~~~~sql
--> What percentage of resitents do not reach their neares station in 15 minutes?
SELECT (
	SELECT SUM(nodes_tiles.einwohner)
	FROM moessingen_2po_4pgr_vertices_pgr
	JOIN nodes_tiles ON moessingen_2po_4pgr_vertices_pgr.id=nodes_tiles.id
	WHERE moessingen_2po_4pgr_vertices_pgr.cost > 0.25
) / (
	SELECT SUM(einwohner)
	FROM nodes_tiles
) * 100 AS percentage;
~~~~

~~~~sql
--> How many people live within the catchment area of the station 'Bodenhausen'?
--> How many pople have 'Bodenhausen' as their closest station?
SELECT SUM(nodes_tiles.einwohner)
FROM moessingen_2po_4pgr_vertices_pgr 
JOIN nodes_tiles ON moessingen_2po_4pgr_vertices_pgr.id=nodes_tiles.id
WHERE moessingen_2po_4pgr_vertices_pgr.station = 'Bodelshausen';
~~~~

~~~~sql
--> Create table that contains the polygons of a voronoi partitioning of all nodes
CREATE TABLE IF NOT EXISTS nodes_voronoi (
   node_id integer NOT NULL,
   geom geometry(POLYGON,4326) NOT NULL,
   PRIMARY KEY (node_id),
   FOREIGN KEY (node_id)
      REFERENCES moessingen_2po_4pgr_vertices_pgr (gid)
);
INSERT INTO nodes_voronoi (
	SELECT nodes.gid AS node_id, voronoi.geom AS geom
	FROM moessingen_2po_4pgr_vertices_pgr AS nodes
	JOIN (
		SELECT (ST_Dump(ST_VoronoiPolygons(ST_Collect(geom)))).geom AS geom
		FROM moessingen_2po_4pgr_vertices_pgr
		WHERE station IS NOT NULL
	) AS voronoi ON ST_Contains(voronoi.geom, nodes.geom)
	WHERE nodes.station IS NOT NULL
);
~~~~

~~~~sql
--> What are the catchment areas of each station?
SELECT nodes.station_id AS station_id, nodes.station AS station_name, ST_Union(nodes_voronoi.geom) AS catchment_area
FROM moessingen_2po_4pgr_vertices_pgr AS nodes
INNER JOIN nodes_voronoi ON nodes.gid=nodes_voronoi.node_id
GROUP BY nodes.station, nodes.station_id;
~~~~

~~~~sql
--> What station is closest to the 'Mühlgärtle' park in Mössingen (coordinates 9.059809°E, 48.407550°N)?
SELECT station
FROM moessingen_2po_4pgr_vertices_pgr
ORDER BY ST_Distance(geom, ST_GeomFromText('POINT(9.059809 48.407550)', 4326)) ASC
LIMIT 1
~~~~

~~~~sql
--> What station is closest to the public swimming pool in Mössingen (coordinates 9.067938°E 48.413985°N)?
SELECT station
FROM moessingen_2po_4pgr_vertices_pgr
ORDER BY ST_Distance(geom, ST_GeomFromText('POINT(9.067938 48.413985)', 4326)) ASC
LIMIT 1
~~~~

~~~~sql
--> Which station has the most people in its catchment area?
SELECT haltepunkte_srid_4326.name, SUM(einwohner), haltepunkte_srid_4326.geom
FROM haltepunkte_srid_4326 
JOIN moessingen_2po_4pgr_vertices_pgr ON moessingen_2po_4pgr_vertices_pgr.station_id=haltepunkte_srid_4326.gid
JOIN nodes_tiles ON moessingen_2po_4pgr_vertices_pgr.id=nodes_tiles.id
GROUP BY haltepunkte_srid_4326.gid
ORDER BY "sum" DESC
LIMIT 1
~~~~
