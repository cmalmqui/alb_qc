# QC ALB QGIS Process Documentation
This workflow will generate a set of rasters needed to evaluate an ALB point cloud. 
The workflow is based on QGIS >= 3.36

Abbrivations: 
    QPT = QGIS Processing Toolbox (Right hand side panel)
    QGM = QGIS Menu Bar (top)


## Fetch the Delivery

- Copy delivery folders to a fast internal disk
- LiDAR loves NVMe M.2 SSDs
- Make sure that LAZ files are 'whitelisted' by the viruscontrol. 

## Create QC Folders

- `/QC`
- `/QC/COPL`
- `/QC/GRID`

## Steps

### 1. Create COPC

For faster operations and visualization we generate a cloud optimized version of the point cloud
  - Tool = **QPT | Point cloud data management | Create COPC**
  - Select laz files
  - Set output directory to `/QC/COPL/`
  - ...wait...

### 2. Build Virtual Point Cloud (VPC)

Generate one 'top file' referencing to each sub tile.
  - Tool = **QPT | Point cloud data management | Build virtual point cloud (VPC)**
  - Select las files from `/QC/COPL/`
  - Enable calculate boundary polygons
  - Enable calculate statistics
  - Enable build overview point cloud
  - Set output file `/QC/alb_copc_pointcloud.vpc`
  - Enable open output file after running algorithm
  - ...wait...

### 3. Build Topobaty DTM

Basic terrain model consisting of 'ground' and 'seabed'.
Note that we do no interpolation in order to show data voids.
  - Tool = **QPT | Point cloud conversion > Export to raster**
  - Select VPC as input
  - Select Z as Attribute
  - Set Resolution to 1
  - Advanced Parameters | Filter expression: `Classification = 2 OR Classification = 40`
  - Set Exported to `/QC/topobathy_1m_dtm.tif`

### 4. Build Watersurface

Generate a terrain model consisting of ground and water surface.
The water surface is generated with interpolation of data voids to ensure that all bathy point has a corresponding surface elevation.
  - Tool = **QPT | Point cloud conversion > Export to raster (using triangulation)**
  - Select VPC as input
  - Select Z as Attribute
  - Set Resolution to 1
  - Advanced Parameters | Filter expression: `Classification = 2 OR Classification = 41`
  - Set Exported to `/QC/topobathy_1m_watersurface.tif`

### 5. Calculate Water Depth

- Tool = **QGM | Raster > Raster Calculator**
- Raster Calculation Expression: `"topobathy_1m_watersurface@1" - "topobathy_1m_dtm@1"`
- Save Result Layer to `/QC/waterdepth_1m.tif`

### 6. Generate Depth Contours

- Tool = **QGM | Raster > Extraction > Contours**
- Input: `/QC/waterdepth_1m.tif`
- Interval: 1m
- Attribute name: Depth
- Enable Produce 3D vector
- Save file to `/QC/waterdepth_1m.shp`

### 7. Calculate Density

- Tool = **QPT | Point cloud extraction > Density**
- Set resolution = 2
- Set Filter expression = `Classification = 2 OR Classification = 40`
- Set Density to `/QC/topobathy_2m_density.tif`

### 8. Calculate Density Binary Plot at 2m Depth

- 5p Threshold for 2x2 grid = 20p
- Tool = **QGM | Raster > Raster Calculator**
- Raster Calculation Expression: `if (("topobathy_2m_density@1" >= 20) AND (("waterdepth_1m@1"  >=  1.5) AND ("waterdepth_1m@1"  <=  2.5)),1,0)`
- Save Result Layer to `/QC/topobathy_2m_density_OK_at_2mDepth.tif`

### 9. Calculate Density Binary Plot 10m Aggregated

- 80% of all 2x2 cells > 20p = 1 else 0

#### 9.1 Calculate 2x2 Binary Raster

- Tool = **QGM | Raster > Raster Calculator**
- Raster Calculation Expression: `if (("topobathy_2m_density@1" >= 20),1,0)`
- Save Result Layer to `/QC/topobathy_2m_density_OK.tif`

#### 9.2 Generate Reference Raster

- Tool = **QGM | Raster Creation > Create constant raster layer**
- Extent: `/QC/topobathy_2m_density_OK_at_2m.tif`
- Pixel Size: 10
- Constant Value: 1
- Integer Output
- Save Result Layer to `/QC/tmp_10mRefGrid.tif`

#### 9.3 Calculate Cell Statistics

- Tool = **QGM | Raster Analysis > Cell statistics**
- Input: `/QC/topobathy_2m_density_OK.tif`
- Statistics: Count
- Ref Layer: `/QC/tmp_10mRefGrid.tif`
- Save
