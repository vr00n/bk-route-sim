# NYC School Bus Route Simulator (Brooklyn)

## Description

This web application simulates hypothetical school bus routes within Brooklyn, NYC. It generates random pickup and school locations within defined constraints, calculates route details using the Open Source Routing Machine (OSRM) API, simulates the timing based on specified parameters, evaluates route feasibility against set criteria, and calculates various heuristic metrics to characterize each route. The results are displayed in a table and visualized on an interactive map and a parallel coordinates plot.

The primary goal is to provide a tool for exploring how different route characteristics (distance, turns, directness, highway usage, etc.) might correlate with operational feasibility under specific time constraints.

## Features

* **Route Simulation:** Generates random routes with a configurable number of pickups and schools.
* **Geographic Constraint:** Routes (pickups/schools) are generated within smaller, random sub-regions of Brooklyn to encourage geographic clustering.
* **Distance Constraint:** Attempts to generate routes with a total driving distance under a specified limit (e.g., 45 miles).
* **OSRM Integration:** Uses the public OSRM API (`router.project-osrm.org`) to calculate realistic driving times, distances, and route geometry based on the OpenStreetMap road network.
* **Timing Simulation:** Models route timing including:
    * Depot departure.
    * Travel time between stops (from OSRM).
    * Configurable wait times at pickups.
    * First pickup time window constraints (e.g., 6 AM - 7 AM).
* **Feasibility Check:** Evaluates each generated route against defined operational criteria.
* **Heuristic Calculation:** Calculates several metrics for each route, including:
    * Directness Ratio
    * Turns per Mile
    * Depot to First Pickup Distance (miles)
    * Average Segment Distance (miles)
    * Stops per Mile
    * % Highway Travel (proxy based on segment speed)
    * % Non-Highway Travel (proxy)
    * Total Stops
    * Total Time (minutes)
    * Total Distance (miles)
* **Visualization:**
    * Displays routes on a Leaflet map, color-coded by feasibility.
    * Shows all generated (valid) routes on an interactive parallel coordinates plot using Plotly.js, allowing exploration of relationships between heuristics and feasibility.
* **Tabular Results:** Presents detailed metrics and feasibility status for each route in a sortable table.

## How it Works

1.  **Initialization:** User sets the number of routes to simulate and clicks "Start Simulation".
2.  **Route Generation Loop:** For each requested route:
    * **Attempt Loop:** Tries up to `MAX_GENERATION_ATTEMPTS` times to generate a valid route under the distance limit.
        * A random sub-region within Brooklyn is defined.
        * A random number of pickup and school locations are generated within this sub-region.
        * An OSRM API call is made to get the route path, distance, duration, steps, and annotations for the sequence: Depot -> Pickups -> Schools.
        * The total distance is checked against `MAX_ROUTE_DISTANCE_MILES`. If it exceeds the limit, a new attempt is made (new sub-region, new points).
    * If a valid route (under distance limit, no OSRM error) is generated within the attempts:
        * **Timing Simulation:** Calculates arrival and departure times at each stop, considering wait times and the first pickup time window.
        * **Feasibility Check:** Compares calculated times against the feasibility criteria.
        * **Heuristic Calculation:** Computes all defined metrics using OSRM output (distance, duration, steps, annotations). The Highway % is estimated based on average speed per detailed route segment from annotations.
        * **UI Update:** Adds the route data to the results table, draws the route path on the Leaflet map (colored by feasibility), and stores the data for the final plot.
3.  **Final Plot:** After all routes are processed, the collected data is used to generate the parallel coordinates plot showing the distribution of metrics across all valid routes.

## Simulation Parameters

* **Depot Location:** Fixed at `40.7283° N, -73.9405° W` (approx. Greenpoint/Williamsburg border).
* **Routing Bounds:** Pickups/Schools generated within random sub-regions inside Brooklyn (approx. `40.57°N` to `40.74°N`, `-74.04°W` to `-73.83°W`).
* **Sub-Region Size:** Approx. 5.5 miles (height) x 5.3 miles (width) - defined by `SUBREGION_HEIGHT_DEG`, `SUBREGION_WIDTH_DEG`.
* **Pickups per Route:** Randomly between 5 and 10 (`MIN_PICKUPS`, `MAX_PICKUPS`).
* **Schools per Route:** Randomly between 1 and 7 (`MAX_SCHOOLS`).
* **First Pickup Window:** Between 6:00 AM and 7:00 AM (`FIRST_PICKUP_START_HOUR`, `FIRST_PICKUP_END_HOUR`).
* **School Start Times:** Randomly between 8:00 AM and 9:00 AM for each school (`SCHOOL_START_HOUR_MIN`, `SCHOOL_START_HOUR_MAX`).
* **Wait Times:**
    * First Pickup: 1-3 minutes (`FIRST_PICKUP_WAIT_MIN_S`, `FIRST_PICKUP_WAIT_MAX_S`).
    * Other Pickups: 1 minute (`OTHER_WAIT_S`).
    * Schools: 0 minutes (`SCHOOL_WAIT_S`).
* **Max Route Distance:** 45 miles (`MAX_ROUTE_DISTANCE_MILES`). Routes exceeding this are discarded.
* **Max Generation Attempts:** 10 (`MAX_GENERATION_ATTEMPTS`) per route index.

## Feasibility Criteria

A generated route (that meets the distance constraint) is considered **Feasible** if **BOTH** of the following conditions are met:

1.  **First School On-Time Arrival:** The bus arrives at the *first school* in its sequence no later than 20 minutes (`MAX_ARRIVAL_DELAY_MINS`) after that school's randomly assigned start time.
2.  **Maximum Ride Time for First Pickup:** The total time elapsed between the bus *departing* the first pickup stop and *arriving* at the first school stop does not exceed 90 minutes (`MAX_FIRST_PICKUP_TRAVEL_TIME_MINS`).

If either condition is not met, the route is marked as "Not Feasible". Routes that fail generation due to the distance limit are marked as "Generation Failed".

## Heuristics Calculated

* **Directness Ratio:** Sum of straight-line distances between consecutive stops / Actual route distance. (Closer to 1 is more direct).
* **Turns/Mile:** Total number of significant turns (excludes slight/sharp) / Actual route distance in miles.
* **Depot-First (mi):** Driving distance from Depot to the first pickup stop.
* **Avg Seg (mi):** Average driving distance of route legs between consecutive pickup/school stops (excludes depot-to-first leg).
* **Stops/Mile:** Total number of pickups and schools / Actual route distance in miles.
* **% Hwy:** Percentage of route distance traveled on segments classified as "highway-like" (based on average speed >= `HIGHWAY_SPEED_THRESHOLD_MPS`).
* **% Non-Hwy:** 100% - `% Hwy`.
* **Stops:** Total number of pickups and schools.
* **Actual Time (min):** Total simulated time from depot departure to arrival at the last stop, including wait times.
* **Actual Dist (mi):** Total driving distance calculated by OSRM.

## Technology Stack

* HTML
* CSS (Tailwind CSS via CDN)
* JavaScript (Vanilla)
* Leaflet.js (for interactive map)
* Plotly.js (for parallel coordinates plot)
* OSRM API (Public Demo Server)

## Limitations & Assumptions

* **OSRM Public Server:** Relies on the free, public OSRM demo server (`router.project-osrm.org`). This server has usage limits and is not intended for heavy load or commercial use. Rate limiting errors (429) may occur if simulations are run too rapidly or frequently.
* **Highway % Proxy:** The classification of road segments as "highway" vs. "non-highway" is an *approximation* based on the average speed calculated from OSRM annotations. It does not use official road classifications and the speed threshold (`HIGHWAY_SPEED_THRESHOLD_MPS`) is a heuristic value.
* **Randomness:** Stop locations are generated randomly within sub-regions, and timings (school start, first pickup) are random within windows. This provides variability but doesn't model real-world demand patterns or specific neighborhood constraints.
* **Traffic:** The OSRM demo server uses traffic data based on OpenStreetMap, but it might not reflect real-time, highly specific NYC traffic conditions perfectly. Route durations are estimates.
* **Sub-Region Generation:** While encouraging clustering, the random sub-region placement might occasionally create unrealistic scenarios (e.g., a dense cluster requiring crossing a major barrier multiple times).
* **No Vehicle Capacity:** The simulation does not model bus capacity limits.
* **Simplified Timing:** Assumes fixed wait times and doesn't account for variability in pickup/dropoff durations.

## How to Run

1.  Save the code as an HTML file (e.g., `route_simulator.html`).
2.  Open the HTML file in a modern web browser (like Chrome, Firefox, Edge).
3.  Enter the desired number of routes to simulate.
4.  Click the "Start Simulation" button.
5.  Observe the results in the table, map, and plot as the simulation progresses.

