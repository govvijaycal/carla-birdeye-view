# Carla Birdeye View Notes

---
## __init__.py
  * Main workhorse with the BirdView, BirdEyeProducer classes.
  * Cropping types include FRONT_AREA_ONLY or FRONT_AND_REAR_AREA variants.
  * Definition of BirdViewMasks enum class and colormap:
    * PEDESTRIANS = 8
    * RED_LIGHTS = 7
    * YELLOW_LIGHTS = 6
    * GREEN_LIGHTS = 5
    * AGENT = 4
    * VEHICLES = 3
    * CENTERLINES = 2
    * LANES = 1
    * ROAD = 0

### `BirdViewProducer` class
  * Rendering area found by taking default size and cropping type and finding a containing square region to render.  This may require adjustment for schemes like nuScenes/Lyft.  The rendering size can be used as a filter to not need to plot all agents in the scene.
  * Roads, lanes, centerlines layers are cached on initialization by saving a numpy array for the entire map.
  * `produce`: does both static + dynamic mask generation cropped around ego.  The static mask is centered around ego with shape based on the rendering_area.  The dynamic mask is also made for the rendering area only, so both masks have the same shape pre-crop.  The masks are then cropped to the final region of interest via `apply_agent_following_transformations_to_masks`.
  * `as_rgb`: Takes all the binary masks and then makes a single RGB image.  It proceeds in the order defined above so that road/lanes are foreground and vehicles/pedestrians/lights are foreground.  Colors defined by the colormap in colors.py.
  * `_render_actors_masks`:  Calls the mask generation for all dynamic agents (traffic lights, pedestrians, vehicles).
  * `apply_agent_following_transformation_to_masks`: Performs an oriented crop is done about the rendering area which orients ego with the vertical image direction and then cuts down to the final BirdViewCropType via a slicing operation.

---
## __main__.py
  * This is what is run when you type `python -m carla-birdeye-view`.
  * Mostly useful to understand how a client application can call the code, relevant lines are:
    ```python
    birdview_producer = BirdViewProducer(
        client,
        PixelDimensions(width=DEFAULT_WIDTH, height=DEFAULT_HEIGHT),
        pixels_per_meter=4,
        crop_type=BirdViewCropType.FRONT_AND_REAR_AREA,
        render_lanes_on_junctions=False,
    )
    birdview: BirdView = birdview_producer.produce(agent_vehicle=agent)
    bgr = cv.cvtColor(BirdViewProducer.as_rgb(birdview), cv.COLOR_BGR2RGB) 
    ```
---

## actors.py
  * Interfaces with the carla client to extract all actors from a snapshot.
  * Breaks down into vehicle, pedestrian, traffic light actors (dynamic).
---

## cache.py
  * Simple helper function to get a hash for a carla map.
---

## colors.py
  * Simple struct class to contain color definitions for RGB visualization.
---

## lanes.py
  * Adapted from the Learning By Cheating codebase.
  * Handles drawing the lane boundaries, including whether it's a solid or broken line.
  * May be worth skipping if centerlines are given instead.
---

## mask.py
  * Contains typedefs and utils related to geometry and bounding boxes.  Main class located here is `MapMaskGenerator`.

### `MapMaskGenerator`:
  * Interfaces with CARLA map API to get topology and waypoints (2m resolution).
  * Map size and corresponding pixel dimensions are found by finding max/min of all waypoints and adding +/- 300 m buffer for dynamic actors.  Seems like this could be shrunken if we know where the ego agent will be located relative to the waypoints.
  * Concept of a rendering window used by BirdViewProducer to 
  * Each mask is simply a H x W binary image (**0 = not present, 1 = present**).
  * `location_to_pixel`: Takes in a carla.Location and returns its pixel location in the mask.  This is implemented so that top-left corner at (0,0) is the min_x and min_y location of the map.
  * `_generate_road_waypoints`: makes a list of waypoints for each road segment in topology with a resolution of 5 cm.  Interesting logic about handling intersection cases where the next waypoint returns multiple options, worth checking again.
  * `road_mask`: draws the road by filling the polygon between the left and right lane boundaries with queried lane width.  There is cv.polylines call that could be removed - it seems like cv.fillPoly should be sufficient.
  * `lanes_mask`: Drwas the left/right lane for each waypoint identified using `_generate_road_waypoints`.  Option to not do this at junctions.
  * `centerlines_mask`: This simply takes the center location of waypoints identified using `_generate_road_waypoints` and uses cv.polylines to draw a layer.  Instead of doing this for all road segments and caching, it could be modified to render only for local segments in view with a given ego_yaw angle to color the mask.  Also an interesting question is what thickness should be used, default is set to 1 px.
  * `agent_vehicle_mask` and `vehicles_mask`: Handle oriented bounding box plotting with cv.fillPoly for ego vehicle/agent and other vehicles.  It assumes the ego vehicle has the "role_name" as "hero" to filter it out.
  * `pedestrians_mask`: Ditto to `vehicles_mask` but for pedestrians.
  * `traffic_lights_mask`: Filters all traffic lights by their state (red, yellow, green) and separates into their individual masks.  The location is taken just as the center point of the light (reasonable if only single-light intersections).  The radius of the plotted circle is set to a constant 1.2 meters.  Could adapt this as in l5kit to change the centerline color based on the traffic light bounding box, but the size may differ.

---

## Things to consider
  * It is possible to add layers for things like speed/yield/stop signs using the CARLA API.  
  * Crosswalks don't seem to be defined in Carla.  Sidewalks and shoulder are defined in the OpenDrive map files as a part of the road network.
  * Adjust cropping to match the other datasets.  This only implements two defaults with ego positioned either at the bottom or in the center.
  * This doesn't currently handle history.  Seems like you would need the class to keep a history of all active agents to aid with this.  Also can filter based on distance from ego to avoid plotting all possible agents.
  * One limitation is the loss of orientation for centerlines colored by relative yaw, possibly to fix by adding another layer.  A possible solution is to make 2 8-bit images to encode 2^16 divisions of 360 degrees, approx. 0.3 deg resolution.)