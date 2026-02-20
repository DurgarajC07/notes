# 🗺️ Spatial Queries - Geographic and Geometric Operations

> Spatial queries enable storage and querying of geographic data (points, lines, polygons) for location-based services, mapping, and GIS applications. Master PostGIS, geometry types, and spatial indexes.

---

## 📖 1. Concept Explanation

### What are Spatial Data Types?

**Spatial data** represents geographic features (locations, shapes, boundaries) stored as **geometry** or **geography** types.

**Basic Spatial Types:**
```sql
-- POINT: Single location (latitude, longitude)
POINT(-122.4194 37.7749)  -- San Francisco

-- LINESTRING: Path or route
LINESTRING(-122.4194 37.7749, -118.2437 34.0522)  -- SF to LA

-- POLYGON: Area or boundary
POLYGON((
    -122.5 37.8,
    -122.3 37.8,
    -122.3 37.7,
    -122.5 37.7,
    -122.5 37.8  -- Closed loop (first = last)
))

-- MULTIPOINT: Multiple locations
MULTIPOINT((-122.4194 37.7749), (-118.2437 34.0522))

-- MULTILINESTRING: Multiple paths
-- MULTIPOLYGON: Multiple areas (e.g., Hawaii islands)
```

**geometry vs geography (PostGIS):**
```
geometry                    geography
│                           │
├── Planar (flat) ✅       ├── Spherical (earth) ✅
├── Faster queries ✅       ├── Accurate distances ✅
├── Meters/feet depend ⚠️  ├── Always meters ✅
├── Use for: Small areas   ├── Use for: Global scale ✅
└── SRID typically 4326    └── Always WGS84 (SRID 4326)
```

**Database Support:**
```
SPATIAL SUPPORT
│
├── PostGIS (PostgreSQL): Best support ✅✅✅
├── MySQL: Spatial extensions ✅
├── SQL Server: geometry/geography types ✅
├── Oracle: Oracle Spatial ✅
└── SQLite: SpatiaLite extension ✅
```

---

## 🧠 2. Why It Matters in Real Systems

### Traditional Distance Calculation (Slow)

**❌ Bad: Haversine formula in application code**
```sql
-- Get all restaurants within 5km
SELECT * FROM restaurants
WHERE (
    6371 * ACOS(
        COS(RADIANS(37.7749)) * COS(RADIANS(latitude)) *
        COS(RADIANS(longitude) - RADIANS(-122.4194)) +
        SIN(RADIANS(37.7749)) * SIN(RADIANS(latitude))
    )
) < 5;

-- Problems:
-- - Calculates distance for EVERY row ❌
-- - No index support ❌
-- - Time: 15s on 1M rows ❌
-- - Complex formula (maintenance burden) ❌
```

**✅ Good: PostGIS with spatial index**
```sql
-- Create geography column + spatial index
ALTER TABLE restaurants ADD COLUMN location geography(POINT, 4326);
UPDATE restaurants SET location = ST_MakePoint(longitude, latitude)::geography;
CREATE INDEX idx_location ON restaurants USING GIST (location);

-- Fast proximity query
SELECT * FROM restaurants
WHERE ST_DWithin(
    location,
    ST_MakePoint(-122.4194, 37.7749)::geography,
    5000  -- 5km in meters
);

-- Benefits:
-- - Uses spatial index ✅
-- - Time: 0.2s on 1M rows ✅ (75x faster)
-- - Built-in functions ✅
-- - Accurate geodesic distance ✅
```

---

## ⚙️ 3. Internal Working

### GIST Index for Spatial Data

```sql
-- GIST (Generalized Search Tree)
CREATE INDEX idx_location_gist ON restaurants USING GIST (location);

-- Index structure: R-Tree
-- Hierarchical bounding boxes:
--
-- Level 0: [Entire world]
--          ├── [North America]
--          │   ├── [California]
--          │   │   ├── [San Francisco]  ← Target area
--          │   │   └── [Los Angeles]
--          │   └── [New York]
--          └── [Europe]
--
-- Query: "Restaurants within 5km of SF"
-- 1. Check Level 0: North America box intersects ✅
-- 2. Check Level 1: California box intersects ✅
-- 3. Check Level 2: SF box intersects ✅ → Return candidates
-- 4. Skip: LA, NY, Europe (no intersection)
-- 5. Exact distance check on candidates
--
-- Time: O(log N) tree traversal vs O(N) sequential scan
```

---

## ✅ 4. Best Practices

### Use geography for Earth-Scale Distances

```sql
-- ❌ geometry: Inaccurate for large distances
CREATE TABLE cities (
    name VARCHAR(100),
    location geometry(POINT, 4326)  -- Planar math ❌
);

SELECT ST_Distance(
    ST_MakePoint(-122.4194, 37.7749),  -- San Francisco
    ST_MakePoint(-0.1278, 51.5074)     -- London
);
-- Returns: ~142 degrees (meaningless!) ❌

-- ✅ geography: Accurate geodesic distance
CREATE TABLE cities (
    name VARCHAR(100),
    location geography(POINT, 4326)  -- Spherical math ✅
);

SELECT ST_Distance(
    ST_MakePoint(-122.4194, 37.7749)::geography,
    ST_MakePoint(-0.1278, 51.5074)::geography
);
-- Returns: 8634239.67 meters (~8,634 km) ✅ Accurate!
```

### Always Create Spatial Index

```sql
-- ✅ GIST index for spatial queries
CREATE INDEX idx_location_gist ON stores USING GIST (location);

-- Query performance:
-- Without index: 12s (sequential scan)
-- With index: 0.3s ✅ (40x faster)
```

### Validate Geometry

```sql
-- ❌ Invalid geometry (self-intersecting polygon)
-- Polygon crosses itself: ><

-- ✅ Check validity
SELECT ST_IsValid(geom) FROM parcels;

-- Fix invalid geometries
UPDATE parcels
SET geom = ST_MakeValid(geom)
WHERE NOT ST_IsValid(geom);
```

---

## ❌ 5. Common Mistakes

### Mistake 1: Longitude, Latitude Order Confusion

```sql
-- ❌ Wrong: Latitude, Longitude
ST_MakePoint(37.7749, -122.4194)  -- latitude first ❌
-- Creates point in Indian Ocean!

-- ✅ Correct: Longitude, Latitude
ST_MakePoint(-122.4194, 37.7749)  -- longitude, latitude ✅
-- San Francisco

-- Mnemonic: "X before Y" (longitude is X, latitude is Y)
```

### Mistake 2: Mixing geometry and geography

```sql
-- ❌ Type mismatch
SELECT ST_Distance(
    location,  -- geography
    ST_MakePoint(-122.4194, 37.7749)  -- geometry
);
-- ERROR: function ST_Distance(geography, geometry) does not exist

-- ✅ Cast to same type
SELECT ST_Distance(
    location,  -- geography
    ST_MakePoint(-122.4194, 37.7749)::geography  -- cast ✅
);
```

### Mistake 3: Not Using ST_DWithin for Radius Queries

```sql
-- ❌ Slow: Calculate distance for all rows
SELECT * FROM stores
WHERE ST_Distance(location, ST_MakePoint(-122.4, 37.7)::geography) < 5000;
-- Can't use index efficiently ❌

-- ✅ Fast: ST_DWithin uses index
SELECT * FROM stores
WHERE ST_DWithin(
    location,
    ST_MakePoint(-122.4, 37.7)::geography,
    5000
);
-- Uses GIST index ✅
```

---

## 🔐 6. Security Considerations

### Sanitize User Location Inputs

```sql
-- ❌ SQL injection risk
DECLARE @lat VARCHAR(20) = @UserInput;
EXECUTE('SELECT * FROM stores WHERE ST_DWithin(location, ST_MakePoint(' + @lon + ', ' + @lat + ')::geography, 5000)');

-- ✅ Use parameterized queries
PREPARE nearby_stores AS
SELECT * FROM stores
WHERE ST_DWithin(location, ST_MakePoint($1, $2)::geography, $3);

EXECUTE nearby_stores(-122.4194, 37.7749, 5000);
```

---

## 🚀 7. Performance Optimization

### Spatial Index Benchmark

```sql
-- Dataset: 1 million store locations
-- Query: Find stores within 5km of point

-- No index: Sequential scan
SELECT * FROM stores
WHERE ST_DWithin(location, ST_MakePoint(-122.4, 37.7)::geography, 5000);
-- Time: 18 seconds ❌

-- With GIST index:
CREATE INDEX idx_location ON stores USING GIST (location);
-- Time: 0.4 seconds ✅ (45x faster)
```

### Use geometry for Small Areas (Performance)

```sql
-- geography: Accurate but slower
SELECT * FROM stores
WHERE ST_DWithin(
    location,  -- geography
    ST_MakePoint(-122.4, 37.7)::geography,
    5000
);
-- Time: 0.4s

-- geometry: Faster for small areas (city-scale)
SELECT * FROM stores
WHERE ST_DWithin(
    location::geometry,  -- Convert to geometry
    ST_Transform(ST_SetSRID(ST_MakePoint(-122.4, 37.7), 4326), 3857),  -- Web Mercator
    5000  -- meters (in projection units)
);
-- Time: 0.15s ✅ (2.6x faster)

-- Use geometry when:
-- - Area < 1000km²
-- - Performance critical
-- - Distance accuracy ±1% acceptable
```

---

## 🧪 8. Examples

### PostGIS: Setup and Basic Operations

```sql
-- Install PostGIS extension
CREATE EXTENSION postgis;

-- Create table with geography column
CREATE TABLE stores (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    location geography(POINT, 4326)
);

-- Insert locations
INSERT INTO stores (name, location) VALUES
('Store A', ST_MakePoint(-122.4194, 37.7749)::geography),  -- San Francisco
('Store B', ST_MakePoint(-118.2437, 34.0522)::geography),  -- Los Angeles
('Store C', ST_MakePoint(-73.9352, 40.7306)::geography);   -- New York

-- Create spatial index
CREATE INDEX idx_location ON stores USING GIST (location);
```

### Distance Calculations

```sql
-- Distance between two points (meters)
SELECT ST_Distance(
    ST_MakePoint(-122.4194, 37.7749)::geography,  -- San Francisco
    ST_MakePoint(-118.2437, 34.0522)::geography   -- Los Angeles
) as distance_meters;
-- Returns: 559120.56 (559km)

-- Find nearest store to location
SELECT 
    name,
    ST_Distance(location, ST_MakePoint(-122.4, 37.75)::geography) as distance_m
FROM stores
ORDER BY location <-> ST_MakePoint(-122.4, 37.75)::geography
LIMIT 1;
-- <-> operator uses index for K-nearest-neighbor ✅
```

### Radius/Proximity Queries

```sql
-- Stores within 10km
SELECT name, ST_AsText(location)
FROM stores
WHERE ST_DWithin(
    location,
    ST_MakePoint(-122.4194, 37.7749)::geography,
    10000  -- 10km in meters
);

-- Stores within 50 miles (convert to meters)
SELECT name
FROM stores
WHERE ST_DWithin(
    location,
    ST_MakePoint(-122.4194, 37.7749)::geography,
    50 * 1609.34  -- 50 miles * meters per mile
);
```

### Polygon Operations

```sql
-- Create delivery zone (polygon)
CREATE TABLE delivery_zones (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    boundary geography(POLYGON, 4326)
);

-- Insert polygon (San Francisco area)
INSERT INTO delivery_zones (name, boundary) VALUES (
    'SF Downtown',
    ST_GeogFromText('POLYGON((
        -122.42 37.78,
        -122.38 37.78,
        -122.38 37.76,
        -122.42 37.76,
        -122.42 37.78
    ))')
);

-- Check if point is within polygon
SELECT EXISTS (
    SELECT 1 FROM delivery_zones
    WHERE ST_Covers(boundary, ST_MakePoint(-122.40, 37.77)::geography)
) as is_in_zone;
-- Returns: true

-- Find all stores in delivery zone
SELECT s.name
FROM stores s
JOIN delivery_zones dz ON ST_Covers(dz.boundary, s.location)
WHERE dz.name = 'SF Downtown';
```

### Bounding Box Queries

```sql
-- Create bounding box (faster than distance for rectangular areas)
SELECT name
FROM stores
WHERE location && ST_MakeEnvelope(
    -122.5, 37.7,  -- min_lon, min_lat (southwest)
    -122.3, 37.8,  -- max_lon, max_lat (northeast)
    4326
)::geography;

-- && operator: Bounding box intersection (very fast with index)
-- Use for initial filter, then ST_DWithin for exact distance
```

### MySQL: Spatial Queries

```sql
-- MySQL 8.0+ has improved spatial support
CREATE TABLE stores (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255),
    location POINT NOT NULL SRID 4326
);

-- Add spatial index
CREATE SPATIAL INDEX idx_location ON stores(location);

-- Insert point
INSERT INTO stores (name, location) VALUES
('Store A', ST_PointFromText('POINT(-122.4194 37.7749)', 4326));

-- Distance query (MySQL 8.0.16+)
SELECT 
    name,
    ST_Distance_Sphere(location, ST_PointFromText('POINT(-122.4 37.75)', 4326)) as distance_m
FROM stores
WHERE ST_Distance_Sphere(location, ST_PointFromText('POINT(-122.4 37.75)', 4326)) < 10000
ORDER BY distance_m;
```

### SQL Server: Spatial Queries

```sql
-- Create table with geography type
CREATE TABLE stores (
    id INT PRIMARY KEY,
    name VARCHAR(255),
    location geography
);

-- Create spatial index
CREATE SPATIAL INDEX idx_location
ON stores(location)
WITH (GRIDS = (MEDIUM, MEDIUM, MEDIUM, MEDIUM));

-- Insert point
INSERT INTO stores VALUES (
    1,
    'Store A',
    geography::Point(37.7749, -122.4194, 4326)  -- lat, lon for SQL Server!
);

-- Distance query
SELECT name
FROM stores
WHERE location.STDistance(geography::Point(37.75, -122.4, 4326)) < 10000;  -- meters

-- Point-in-polygon
DECLARE @zone geography;
SET @zone = geography::STPolyFromText('POLYGON((...))', 4326);

SELECT name
FROM stores
WHERE @zone.STContains(location) = 1;
```

---

## 🏗️ 9. Real-World Use Cases

### Case Study 1: Uber/Lyft - Driver Matching

```sql
-- Find available drivers within 5km of passenger
CREATE TABLE drivers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    location geography(POINT, 4326),
    status VARCHAR(20),  -- 'available', 'busy'
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_driver_location ON drivers USING GIST (location);
CREATE INDEX idx_driver_status ON drivers(status) WHERE status = 'available';

-- Real-time query: Find nearest 5 available drivers
SELECT 
    id,
    name,
    ST_Distance(location, ST_MakePoint(-122.4194, 37.7749)::geography) as distance_m
FROM drivers
WHERE status = 'available'
  AND updated_at > NOW() - INTERVAL '5 minutes'  -- Active in last 5 min
  AND ST_DWithin(
      location,
      ST_MakePoint(-122.4194, 37.7749)::geography,
      5000  -- 5km radius
  )
ORDER BY location <-> ST_MakePoint(-122.4194, 37.7749)::geography  -- K-nearest
LIMIT 5;
```

### Case Study 2: Real Estate - Property Search

```sql
-- Property listings with boundaries
CREATE TABLE properties (
    id SERIAL PRIMARY KEY,
    address TEXT,
    price NUMERIC(12,2),
    bedrooms INT,
    boundary geography(POLYGON, 4326),  -- Property boundary
    location geography(POINT, 4326)     -- Centroid for distance
);

CREATE INDEX idx_property_location ON properties USING GIST (location);
CREATE INDEX idx_property_boundary ON properties USING GIST (boundary);

-- School districts
CREATE TABLE school_districts (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    rating NUMERIC(2,1),
    boundary geography(POLYGON, 4326)
);

CREATE INDEX idx_district_boundary ON school_districts USING GIST (boundary);

-- Search: 3BR homes < $1M within 10km, in good school district
SELECT 
    p.address,
    p.price,
    sd.name as school_district,
    sd.rating,
    ST_Distance(p.location, ST_MakePoint(-122.4, 37.75)::geography) as distance_m
FROM properties p
JOIN school_districts sd ON ST_Covers(sd.boundary, p.location)
WHERE p.bedrooms >= 3
  AND p.price < 1000000
  AND ST_DWithin(
      p.location,
      ST_MakePoint(-122.4, 37.75)::geography,
      10000
  )
  AND sd.rating >= 8.0
ORDER BY distance_m
LIMIT 20;
```

### Case Study 3: Logistics - Delivery Route Optimization

```sql
-- Delivery addresses
CREATE TABLE deliveries (
    id SERIAL PRIMARY KEY,
    address TEXT,
    location geography(POINT, 4326),
    status VARCHAR(20),
    priority INT
);

CREATE INDEX idx_delivery_location ON deliveries USING GIST (location);

-- Warehouse locations
CREATE TABLE warehouses (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    location geography(POINT, 4326)
);

-- Assign deliveries to nearest warehouse
SELECT 
    d.id as delivery_id,
    d.address,
    w.name as assigned_warehouse,
    ST_Distance(d.location, w.location) as distance_m
FROM deliveries d
CROSS JOIN LATERAL (
    SELECT id, name, location
    FROM warehouses
    ORDER BY location <-> d.location
    LIMIT 1
) w
WHERE d.status = 'pending';

-- Group deliveries within 2km for route batching
SELECT 
    ST_ClusterKMeans(location::geometry, 10) OVER () as cluster_id,
    COUNT(*) as deliveries_in_cluster,
    ST_AsText(ST_Centroid(ST_Collect(location::geometry))) as cluster_center
FROM deliveries
WHERE status = 'pending'
GROUP BY cluster_id;
```

### Case Study 4: Store Locator

```sql
-- Store locations with opening hours
CREATE TABLE stores (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    address TEXT,
    location geography(POINT, 4326),
    open_time TIME,
    close_time TIME,
    has_pharmacy BOOLEAN
);

CREATE INDEX idx_store_location ON stores USING GIST (location);

-- Find nearest 3 stores with pharmacy, currently open
SELECT 
    name,
    address,
    ST_Distance(location, ST_MakePoint($1, $2)::geography) / 1609.34 as miles,
    open_time,
    close_time
FROM stores
WHERE has_pharmacy = true
  AND CURRENT_TIME BETWEEN open_time AND close_time
ORDER BY location <-> ST_MakePoint($1, $2)::geography
LIMIT 3;
```

---

## ❓ 10. Frequently Asked Interview Questions

### Q1: What's the difference between geometry and geography types in PostGIS?

**Answer:**

| Feature | geometry | geography |
|---------|----------|-----------|
| **Coordinate System** | Planar (flat) | Spherical (Earth) |
| **Distance Accuracy** | Inaccurate for large areas | Accurate (geodesic) ✅ |
| **Distance Units** | Depends on SRID | Always meters ✅ |
| **Performance** | Faster ✅ (2-3x) | Slower (complex math) |
| **Use Case** | Small areas (<1000km²) | Global scale ✅ |
| **Example** | City mapping | Flight routes |

**Example:**
```sql
-- geometry (planar): Inaccurate for SF to London
SELECT ST_Distance(
    ST_MakePoint(-122.4, 37.7),  -- San Francisco
    ST_MakePoint(-0.1, 51.5)     -- London
);
-- Returns: ~142 degrees ❌ (meaningless!)

-- geography (spherical): Accurate
SELECT ST_Distance(
    ST_MakePoint(-122.4, 37.7)::geography,
    ST_MakePoint(-0.1, 51.5)::geography
);
-- Returns: 8638243 meters (8,638 km) ✅

-- geometry for performance (city-scale)
-- geography for accuracy (global-scale)
```

---

### Q2: How do you find the nearest N locations efficiently?

**Answer:**

**Use K-nearest-neighbor (KNN) operator `<->`:**

```sql
-- ✅ Fast: Uses GIST index with KNN
SELECT 
    name,
    ST_Distance(location, ST_MakePoint(-122.4, 37.7)::geography) as distance_m
FROM stores
ORDER BY location <-> ST_MakePoint(-122.4, 37.7)::geography
LIMIT 5;

-- Execution: Index scan, returns top 5 without scanning all rows ✅
-- Time: 0.05s on 1M rows
```

**❌ Slow: Distance in WHERE (can't use index efficiently):**
```sql
SELECT name, ST_Distance(location, ...) as distance_m
FROM stores
ORDER BY distance_m
LIMIT 5;

-- Execution: Calculates distance for ALL rows first ❌
-- Time: 12s on 1M rows
```

**Combined: KNN + Distance Filter:**
```sql
-- First filter by radius, then sort by distance
SELECT name, ST_Distance(location, point) as distance_m
FROM stores, ST_MakePoint(-122.4, 37.7)::geography as point
WHERE ST_DWithin(location, point, 50000)  -- Within 50km
ORDER BY location <-> point
LIMIT 10;
-- Fastest for large datasets ✅
```

---

### Q3: Explain how spatial indexing works.

**Answer:**

**GIST index uses R-Tree structure:**

**Concept:**
- Hierarchical bounding boxes (rectangles)
- Parent box contains all child boxes
- Query traverses tree, prunes branches early

**Example:**
```
World
├── North America
│   ├── USA
│   │   ├── California
│   │   │   ├── San Francisco ← Target
│   │   │   └── Los Angeles
│   │   └── New York
│   └── Canada
└── Europe
    ├── UK
    └── France
```

**Query: "Stores within 5km of San Francisco"**
```
1. Check World box: Intersects ✅ → Go deeper
2. Check North America: Intersects ✅ → Go deeper
3. Check USA: Intersects ✅ → Go deeper
4. Check California: Intersects ✅ → Go deeper
5. Check San Francisco box: Intersects ✅ → Return candidates
6. Skip Los Angeles, New York (bounding boxes don't intersect)
7. Skip Canada, Europe (pruned at higher level)
8. Exact distance check on SF candidates
```

**Performance:**
- Time: O(log N) tree traversal
- Without index: O(N) sequential scan
- Speedup: 40-100x for radius queries

**Create index:**
```sql
CREATE INDEX idx_location ON stores USING GIST (location);
```

---

### Q4: How do you check if a point is inside a polygon?

**Answer:**

**PostGIS:**
```sql
-- ST_Contains: Polygon contains point (exclusive boundary)
SELECT ST_Contains(
    ST_GeomFromText('POLYGON((
        -122.5 37.8,
        -122.3 37.8,
        -122.3 37.7,
        -122.5 37.7,
        -122.5 37.8
    ))'),
    ST_MakePoint(-122.4, 37.75)
);
-- Returns: true (point is inside)

-- ST_Covers: Polygon covers point (inclusive boundary)
SELECT ST_Covers(polygon, point);
-- Returns: true even if point is on boundary

-- ST_Within: Inverse of ST_Contains
SELECT ST_Within(point, polygon);
```

**Real-world example:**
```sql
-- Check if address is in delivery zone
SELECT EXISTS (
    SELECT 1
    FROM delivery_zones
    WHERE ST_Covers(boundary, customer_location)
) as delivers_here;
```

**Performance tip:**
```sql
-- ✅ Use bounding box filter first (fast)
SELECT name
FROM parcels
WHERE boundary && ST_MakePoint(-122.4, 37.75)::geometry  -- Bounding box (fast)
  AND ST_Contains(boundary, ST_MakePoint(-122.4, 37.75)); -- Exact (slower)

-- && uses index for initial filter, then exact check
-- Much faster than ST_Contains alone on large datasets
```

---

### Q5: What are the common pitfalls when working with spatial data?

**Answer:**

**1. Longitude/Latitude order confusion:**
```sql
-- ❌ Wrong: Latitude first
ST_MakePoint(37.7749, -122.4194)  -- Creates point in wrong location!

-- ✅ Correct: Longitude, Latitude (X, Y)
ST_MakePoint(-122.4194, 37.7749)
```

**2. Mixing geometry and geography:**
```sql
-- ❌ Type mismatch error
SELECT ST_Distance(
    location,  -- geography  
    ST_MakePoint(-122.4, 37.7)  -- geometry
);

-- ✅ Cast to same type
SELECT ST_Distance(
    location,
    ST_MakePoint(-122.4, 37.7)::geography
);
```

**3. Not creating spatial index:**
```sql
-- ❌ Missing index: 50x slower
-- ✅ Add GIST index:
CREATE INDEX idx_location ON table USING GIST (location);
```

**4. Using ST_Distance instead of ST_DWithin for radius:**
```sql
-- ❌ Slow: Calculates exact distance for all rows
WHERE ST_Distance(location, point) < 5000;

-- ✅ Fast: Index-optimized radius check
WHERE ST_DWithin(location, point, 5000);
```

**5. Forgetting SRID (Spatial Reference System):**
```sql
-- ❌ No SRID: Assumes 0 (undefined)
ST_MakePoint(-122.4, 37.7)

-- ✅ Set SRID: 4326 = WGS84 (GPS coordinates)
ST_SetSRID(ST_MakePoint(-122.4, 37.7), 4326)
```

**6. Invalid geometries:**
```sql
-- ✅ Always validate after user input
SELECT ST_IsValid(geom) FROM parcels;

-- Fix invalid geometries
UPDATE parcels SET geom = ST_MakeValid(geom) WHERE NOT ST_IsValid(geom);
```

---

## 🧩 11. Design Patterns

### Pattern 1: Hybrid Indexing (Spatial + B-Tree)

```sql
-- Combine spatial index with regular indexes
CREATE TABLE stores (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    location geography(POINT, 4326),
    category VARCHAR(50),
    rating NUMERIC(2,1),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_location ON stores USING GIST (location);  -- Spatial
CREATE INDEX idx_category_rating ON stores(category, rating);  -- B-tree

-- Query uses both indexes
SELECT name
FROM stores
WHERE category = 'restaurant'
  AND rating >= 4.0
  AND ST_DWithin(location, ST_MakePoint(-122.4, 37.7)::geography, 5000);
-- Optimizer chooses best index combination
```

### Pattern 2: Geohash for Scalability

```sql
-- Pre-compute geohash for filtering
CREATE TABLE stores (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    location geography(POINT, 4326),
    geohash VARCHAR(12)  -- Precision: 12 chars (~1m)
);

-- Populate geohash (using PostGIS extension or application)
UPDATE stores SET geohash = ST_GeoHash(location::geometry, 12);

CREATE INDEX idx_geohash ON stores(geohash);

-- Filter by geohash prefix (fast string match)
SELECT name
FROM stores
WHERE geohash LIKE '9q8yy%'  -- Prefix matches area
  AND ST_DWithin(location, point, 5000);  -- Exact distance
-- Geohash narrows candidates, then exact distance check
```

### Pattern 3: Materialized Bounding Boxes

```sql
-- Store precomputed bounding box for complex polygons
CREATE TABLE parcels (
    id SERIAL PRIMARY KEY,
    boundary geometry(POLYGON, 4326),
    bbox geometry(POLYGON, 4326)  -- Simple bounding box
);

-- Populate bounding boxes
UPDATE parcels SET bbox = ST_Envelope(boundary);

CREATE INDEX idx_bbox ON parcels USING GIST (bbox);

-- Two-phase query: bbox filter → exact check
SELECT id
FROM parcels
WHERE bbox && search_area  -- Fast bbox intersection
  AND ST_Intersects(boundary, search_area);  -- Exact (slower)
```

---

## 📚 Summary

### Key Takeaways

1. **Spatial types**: POINT, LINESTRING, POLYGON for geographic data
2. **PostGIS**: Best spatial support (PostgreSQL extension)
3. **geometry vs geography**: Planar (fast) vs spherical (accurate)
4. **GIST index**: R-Tree structure, 40-100x speedup for spatial queries
5. **ST_DWithin**: Radius queries (index-optimized) ✅
6. **KNN operator `<->`**: Nearest neighbor queries (efficient)
7. **Lon/Lat order**: ST_MakePoint(longitude, latitude) — X before Y
8. **ST_Contains/ST_Covers**: Point-in-polygon checks
9. **Common mistake**: Forgetting spatial index (50x slower)
10. **Real-world**: Ride-sharing, real estate, logistics, store locators

**PostGIS Quick Reference:**

```sql
-- Create geography column
ALTER TABLE stores ADD COLUMN location geography(POINT, 4326);

-- Create GIST index
CREATE INDEX idx_location ON stores USING GIST (location);

-- Insert point
INSERT INTO stores (name, location) VALUES
('Store', ST_MakePoint(-122.4194, 37.7749)::geography);

-- Distance query
SELECT ST_Distance(
    ST_MakePoint(-122.4, 37.7)::geography,
    ST_MakePoint(-118.2, 34.0)::geography
) as meters;

-- Radius query
SELECT * FROM stores
WHERE ST_DWithin(location, ST_MakePoint(-122.4, 37.7)::geography, 5000);

-- Nearest N
SELECT * FROM stores
ORDER BY location <-> ST_MakePoint(-122.4, 37.7)::geography
LIMIT 5;

-- Point in polygon
SELECT ST_Contains(polygon, point);
```

---

**Next:** [README.md](README.md) - Advanced SQL overview  
**Related:** [../06_Indexing_Performance/03_Index_Types.md](../06_Indexing_Performance/03_Index_Types.md) - Index strategies

---

*Last Updated: February 20, 2026*
