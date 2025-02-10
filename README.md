# Arrowhead Stadium Traffic Analysis
## Table of Contents
1. [Project Overview](#project-overview)
2. [Technologies Used](#technologies-used)
3. [Data Collection](#data-collection)
4. [Key Features](#key-features)
5. [Step by Step Implementation](#step-by-step-implementation)  
   - [Step 1: Install Required Python Packages](#step-1-install-required-python-packages)  
   - [Step 2: Extracting Camera Locations from KMZ File](#step-2-extracting-camera-locations-from-kmz-file)  
   - [Step 3: Fetching and Visualizing Alternative Routes using Google Maps API](#step-3-fetching-and-visualizing-alternative-routes-using-google-maps-api)  
   - [Step 4: Generating Time-Based Traffic Heatmaps (Optional)](#step-4-generating-time-based-traffic-heatmaps-optional)  
   - [Step 5: Identifying and Mapping Congestion Points](#step-5-identifying-and-mapping-congestion-points)  
   - [Step 6: Integrating Congestion, Routes, Cameras, and DMS into a Combined Traffic Map](#step-6-integrating-congestion-routes-cameras-and-dms-into-a-combined-traffic-map)   
6. [Conclusion](#conclusion)

## Project Overview
This project utilizes Google Maps Directions API to analyze traffic conditions around Arrowhead Stadium during the Chiefs vs. Bills game on 01/26/25 from 12:30 PM to 6:00 PM (CST).

The analysis focuses on:
- Identifying traffic patterns and pinch points on major entry roads.
- Understanding alternative routes suggested by the API.
- Extracting and analyzing camera locations from a KMZ file to enhance ITS recommendations.
- Recommending potential ITS enhancements (Cameras & DMS signs).
- Visualizing congestion trends with an interactive traffic map.

---

## Technologies Used
- **Google Maps Directions API** – Fetches alternative routes and travel times.
- **Python** – Main programming language.
- **Folium** – Generates interactive traffic maps.
- **Pandas** – Processes and analyzes traffic data.
- **Haversine** – Calculates distances between congestion points.
- **Matplotlib** – Creates visualizations of traffic trends.
- **KMZ/KML Parsing (`pykml`, `fastkml`)** – Extracts camera locations from map files.

---

## Data Collection
- **Predefined origins** within a 5-mile radius of Arrowhead Stadium.
- **Main road considerations**: 
  - I-70 (East/West)
  - I-435 (North/South)
- **Data timeframe**: 12:30 PM to 6:00 PM CST (Kickoff: 5:30 PM).
- Google Maps API fetched alternative routes from each direction.

---

## Key Features
- **Traffic Flow Analysis** – Identifies congestion patterns.
- **Congestion Detection** – Flags road segments where speed is below 40 mph.
- **Dynamic Route Mapping** – Visualizes alternative routes using Google Maps API.
- **KMZ Camera Data Extraction** – Identifies existing cameras for ITS planning.
- **Geospatial Visualization** – Pinpoints bottlenecks using interactive maps.
- **ITS Recommendations** – Suggests additional cameras & DMS signs for use during major events.

---
## Step by Step Implementation
To implement this analysis, install the necessary dependencies before execution.

# **Step 1. Install Required Python Packages**
Run the following command in Google Colab, Jupyter Notebook, or Terminal:
```python
pip install pykml fastkml polyline pytz haversine 
```
# **Step 2: Extracting Camera Locations from KMZ File**
This step focuses on extracting camera and DMS loations from a KMZ file containing KML data. The extracted data includes:
- Camera markers
- DMS markers
- Latitude and longitude coordinates
- Filtered locations (excluding Arrowhead Stadium and directional markers)

The extracted data is stored in a Pandas DataFrame and will be used for further analysis in later steps.

### 1.  Extracting KML Data from KMZ
The script extracts KML files from a KMZ archive and parses them using `xml.etree.ElementTree`.
```python
from zipfile import ZipFile
import xml.etree.ElementTree as ET
import pandas as pd

# Paths to your files
kmz_cams_file = "/content/drive/MyDrive/cams.kmz"  # Path to your KMZ file

# Extract the KML file from the KMZ file
kml_extracted_path = "/content/drive/MyDrive/extracted_kml.kml"
with ZipFile(kmz_cams_file, 'r') as kmz:
    # Extract the KML file
    kml_filename = [file for file in kmz.namelist() if file.endswith('.kml')][0]
    kmz.extract(kml_filename, path="/content/drive/MyDrive")
    kml_full_path = f"/content/drive/MyDrive/{kml_filename}"
```
### 2.  Parsing KML to Extract Camera Locations
Once the KML file is extracted, we parse the XML to retrieve camera locations.
```python
# Parse the extracted KML file
tree = ET.parse(kml_full_path)
root = tree.getroot()

# Define the namespace to parse KML properly
namespace = {"kml": "http://www.opengis.net/kml/2.2"}

# Extract placemark names and coordinates
camera_data = []
for placemark in root.findall(".//kml:Placemark", namespace):
    name = placemark.find("kml:name", namespace).text
    coords = placemark.find(".//kml:coordinates", namespace).text.strip()
    lon, lat, _ = map(float, coords.split(","))
    camera_data.append({"Name": name, "Latitude": lat, "Longitude": lon})

# Convert to a DataFrame
camera_df = pd.DataFrame(camera_data)
```

### 3. Filtering Out Unnecessary Locations
Remove any irrelevant placemarks such as Arrowhead Stadium (the destination) and directional markers.
```python
# Remove rows where 'Name' contains "Arrowhead" since that's the actual destination
camera_df = camera_df[~camera_df["Name"].str.contains("|".join(["Arrowhead", "North", "East", "West", "South"]), case=False, na=False)]

# Display the Camera DataFrame
print("Camera  Locations:")
print(camera_df)
```
Repeat the same for DMS locations
```python
# Paths to your files
kmz_dms_file = "/content/drive/MyDrive/dms.kmz"  # Path to your KMZ file

# Extract the KML file from the KMZ file
kml_extracted_path = "/content/drive/MyDrive/extracted_kml.kml"
with ZipFile(kmz_dms_file, 'r') as kmz:
    # Extract the KML file
    kml_filename = [file for file in kmz.namelist() if file.endswith('.kml')][0]
    kmz.extract(kml_filename, path="/content/drive/MyDrive")
    kml_full_path = f"/content/drive/MyDrive/{kml_filename}"

# Parse the extracted KML file to retrieve placemark data
tree = ET.parse(kml_full_path)
root = tree.getroot()

# Define the namespace to parse KML properly
namespace = {"kml": "http://www.opengis.net/kml/2.2"}

# Extract placemark names and coordinates
dms_data = []
for placemark in root.findall(".//kml:Placemark", namespace):
    name = placemark.find("kml:name", namespace).text
    coords = placemark.find(".//kml:coordinates", namespace).text.strip()
    lon, lat, _ = map(float, coords.split(","))
    dms_data.append({"Name": name, "Latitude": lat, "Longitude": lon})

# Convert to a DataFrame
dms_df = pd.DataFrame(dms_data)

# Display the Camera and DMS DataFrame
print("DMS Locations:")
print(dms_df)
```

# Step 3: Fetching and Visualizing Alternative Routes using Google Maps API
In this step, we use the Google Maps Directions API to:
- Fetch alternative driving routes from predefined origins to Arrowhead Stadium.
- Decode and plot the route polylines on an interactive map.
- Visualize the different routes using Folium.
- Identify potential traffic congestion areas by analyzing route options.
---

## Code Implementation
Before running the script, ensure you have installed the necessary dependencies:

```bash
pip install folium requests  polyline openpyxl
```
### 1. Securely Load th API key
To avoid hardcoding sensitive credentials, the API key is stored securely in a file. This function reads the API key stored in a text file and returns it for use in API requests.
```python
import os
import requests
import folium
from polyline import decode as decode_polyline

# Function to load the API key securely from Google Drive
def load_api_key(filepath):
    with open(filepath, 'r') as f:
        for line in f:
            key, value = line.strip().split('=')
            if key == 'api_key':
                return value
```
### 2. Fetching Alternative Routes from Google Maps API
The function below queries the Google Maps Directions API to fetch alternative routes for a given origin and destination
```python
# Fetch route details from Google Maps Directions API
def fetch_routes(api_key, origin, destination):
    url = "https://maps.googleapis.com/maps/api/directions/json"
    params = {
        "origin": origin,
        "destination": destination,
        "key": api_key,
        "alternatives": "true"  # Enable fetching alternative routes
    }
    response = requests.get(url, params=params)
    if response.status_code == 200:
        data = response.json()
        if data['status'] == 'OK':
            # Extract all routes
            routes = []
            for route in data['routes']:
                polyline = route['overview_polyline']['points']
                routes.append(decode_polyline(polyline))  # Decode the polyline into lat/lng points
            return routes
        else:
            print(f"Error from API: {data['status']}")
    else:
        print(f"HTTP Error: {response.status_code}")
    return None
```
How It Works:
1. Sends a request to Google Maps Directions API.
2. Retrieves multiple alternative routes (if available).
3. Decodes the polyline to obtain route coordinates.

### 3. Plotting Routes on a Folium Map
This function plots the alternative routes on an interactive Folium map.
```python
# Plot routes on a map
def plot_routes_on_map(api_key, data):
    # Initialize the map at a central location
    m = folium.Map(location=[39.048786, -94.484566], zoom_start=13)
    colors = ['blue', 'green', 'purple', 'orange', 'red']  # Different colors for multiple routes

    for index, row in data.iterrows():
        origin = f"{row['origin_lat']},{row['origin_lng']}"
        destination = f"{row['destination_lat']},{row['destination_lng']}"

        # Fetch all routes data
        routes = fetch_routes(api_key, origin, destination)
        if routes:
            for i, route_coords in enumerate(routes):
                # Assign different colors for each route
                color = colors[i % len(colors)]
                # Draw the route on the map
                folium.PolyLine(route_coords, color=color, weight=2.5, opacity=0.7,
                                tooltip=f"Route {i + 1}").add_to(m)

            # Add markers for origin and destination
            folium.Marker(
                location=[row['origin_lat'], row['origin_lng']],
                popup=f"Origin: {row['origin']}",
                icon=folium.Icon(color='blue', icon='info-sign')
            ).add_to(m)

            folium.Marker(
                location=[row['destination_lat'], row['destination_lng']],
                popup=f"Destination: {row['destination']}",
                icon=folium.Icon(color='red', icon='info-sign')
            ).add_to(m)

    return m
```
How It Works:
1. Initializes a Folium map centered near Arrowhead Stadium.
2. Iterates through each origin-destination pair in the dataset.
3. Fetches alternative routes for each pair.
4. Plots routes with distinct colors.
5. Adds markers for origin and destination points.

### 4. Main Execution Script
The script loads the dataset, extracts origin-destination pairs, and calls the mapping functions.
```python
# Main script execution
if __name__ == "__main__":
    # Path to the file containing the Google Maps API key
    api_key_path = "/content/drive/MyDrive/API_K.txt"

    # Load the API key
    api_key = load_api_key(api_key_path)

    # Load the dataset (update with actual dataset path)
    dataset_path = "/content/drive/MyDrive/chiefs-bills.xlsx"
    df = pd.read_excel(dataset_path)

    # Parse the `query` column to extract origin and destination coordinates
    df[['origin_lat', 'origin_lng', 'destination_lat', 'destination_lng']] = df['query'].str.split(', ', expand=True)
    df['origin_lat'] = df['origin_lat'].astype(float)
    df['origin_lng'] = df['origin_lng'].astype(float)
    df['destination_lat'] = df['destination_lat'].astype(float)
    df['destination_lng'] = df['destination_lng'].astype(float)

    # Call the function to plot the routes on a map
    traffic_map = plot_routes_on_map(api_key, df)

    # Save the map to an HTML file
    traffic_map.save("traffic_routes_map24.html")
    print("Map saved as traffic_routes_map24.html")
 ```

# Step 4: Generating Time-Based Traffic Heatmaps (Optional)
This step is optional and focuses on creating time-based traffic heatmaps to analyze congestion at specific intervals. Heatmaps help visualize:
- Traffic density along major routes to Arrowhead Stadium.
- Congestion trends by time of day.
- Frequently used roads and hotspots.

## Code Implementation
Ensure that you have installed the following dependencies before running the script:

```bash
pip install folium requests pytz haversine pandas polyline openpyxl
```
### 1. Convert UTC Timestamps to Central Time (CST/CDT)
To ensure time accuracy, we convert UTC timestamps to Central Time.
```python
from datetime import datetime, timedelta

# Convert UTC time to Central Time (CT)
def convert_to_central_time(utc_time):
    # If the input is an integer (Unix timestamp)
    if isinstance(utc_time, int):
        utc_dt = datetime.utcfromtimestamp(utc_time)  # Convert Unix timestamp to datetime
    else:  # If the input is a string
        utc_dt = datetime.strptime(utc_time, "%Y-%m-%d %H:%M:%S")

    # Convert to Central Time (UTC-6)
    central_dt = utc_dt - timedelta(hours=6)
    return central_dt.strftime("%Y-%m-%d %H:%M:%S")
 ```
### 2. Create a Heatmap for Selected Time Intervals
This function filters route data for a specific time range and plots a heatmap of congestion.
```python
import folium
from folium.plugins import HeatMap
import requests
from polyline import decode as decode_polyline

# Create a heatmap based on routes and time intervals
def create_route_heatmap(data, api_key, time_interval):
    # Filter data by the specified time interval
    filtered_data = data[data['central_time'].str.contains('|'.join(time_interval))]

    # Initialize the map
    m = folium.Map(location=[39.048786, -94.484566], zoom_start=13)

    # Extract route coordinates for the heatmap
    heat_data = []
    for index, row in filtered_data.iterrows():
        origin = f"{row['origin_lat']},{row['origin_lng']}"
        destination = f"{row['destination_lat']},{row['destination_lng']}"

        # Fetch all routes data
        routes = fetch_routes(api_key, origin, destination)
        if routes:
            for route_coords in routes:
                for lat, lng in route_coords:
                    heat_data.append([lat, lng, 1])  # Add each point with a weight of 1

    # Add HeatMap layer
    HeatMap(heat_data).add_to(m)

    return m
```
### 3.  Applying Time Conversion and Generating Heatmaps
1. Convert timestamps in the dataset to Central Time.
2. Select a time interval (e.g., 4:30 PM - 5:30 PM) for analysis.
3. Generate and save the heatmap as an interactive HTML file.
```python
# Convert UTC timestamps to Central Time if a timestamp column exists
if 'timestamp' in df.columns:
    df['central_time'] = df['timestamp'].apply(convert_to_central_time)

# Define the time interval for analysis (Example: 4:30 PM - 5:30 PM CST)
time_interval = "16:30", "17:30"

# Create the heatmap for the specified time interval
heatmap = create_route_heatmap(df, api_key, time_interval)

# Save the map to an HTML file
heatmap.save("traffic_route_heatmap4.html")
print("Map saved as traffic_route_heatmap4.html")
```
Expected Output
1. An interactive heatmap visualizing congestion trends during selected time intervals.
2. Darker red areas indicate higher traffic density.
3. The output file traffic_route_heatmap4.html can be opened in a browser

# Step 5: Identifying and Mapping Congestion Points
This step focuses on detecting **congestion points** along key routes leading to **Arrowhead Stadium**. The script:
- **Calculates traffic speeds** based on road segment distances and travel times.
- **Flags congested segments** where speeds fall **below 40 mph**.
- **Finds latitude/longitude coordinates** of congestion points along each route.
- **Maps congestion locations** using **small red dots** on an interactive Folium map.

## Code Implementation
Before running the script, ensure you have the necessary dependencies installed:

```bash
pip install folium pytz haversine pandas requests polyline openpyxl
```
### 1. Convert UTC Time to Central Time (CST/CDT)
Since Google Maps API returns timestamps in UTC, they are converted to local time (CST/CDT).
```python
from datetime import datetime
from pytz import timezone

# Convert UTC time to Central Time (CT)
def convert_to_central_time(utc_time_str):
    utc = timezone('UTC')
    central = timezone('US/Central')
    utc_time = datetime.strptime(utc_time_str, "%m/%d/%Y %H:%M:%S")
    central_time = utc.localize(utc_time).astimezone(central)
    return central_time.strftime("%m/%d/%Y %H:%M:%S")
```
### 2. Identify Congestion Based on Speed Threshold
A speed threshold of 40 mph is used to classify road segments as congested.
```python
# Analyze congestion flags based on speed < 40 mph
def update_congestion_flags(speeds_mph):
    return [speed < 40 if speed is not None else False for speed in speeds_mph]
```
### 3. Find Exact Locations of Congestion Points
Once congestion is detected, route polyline data is used to pinpoint congestion coordinates.
```python
from polyline import decode as decode_polyline
from haversine import haversine, Unit

# Find lat/long of congested points based on distance along polyline
def get_congestion_coordinates(route_polyline, distances_miles):
    route_coords = decode_polyline(route_polyline)
    congestion_coords = []
    total_distance = 0

    for i in range(1, len(route_coords)):
        # Calculate segment distance in miles
        dist = haversine(route_coords[i - 1], route_coords[i])
        total_distance += dist

        # Match congestion distances to coordinates
        for distance in distances_miles:
            if total_distance >= distance:
                congestion_coords.append(route_coords[i])
                distances_miles.remove(distance)
                break
        if not distances_miles:
            break
    return congestion_coords
```
### 4. Process and Analyze Traffic Data
Extracts polylines for all routes.
Calculates speeds using travel times and segment distances.
Applies congestion detection logic.
Finds congestion coordinates for flagged segments.

```python
# Set file paths
api_key_path = "/content/drive/MyDrive/API_K.txt"
dataset_path = "/content/drive/MyDrive/chiefs-bills.xlsx"

# Load the API key
api_key = load_api_key(api_key_path)

# Load the dataset
df = pd.read_excel(dataset_path)

# Extract route polylines
df['origin'] = df['origin_coordinates']
df['destination'] = df['destination_coordinates']
df['polyline'] = df.apply(
    lambda row: fetch_route_polyline(api_key, row['origin'], row['destination']), axis=1
)

# Parse road timing data
df['road_timing_parsed'] = df['road_distance_timing(meters:seconds)'].apply(
    lambda x: json.loads(x.replace("'", '"'))
)

# Convert road timing data and calculate speeds
df['speeds_mph'], df['congestion_flags_40mph'] = zip(
    *df['road_timing_parsed'].apply(
        lambda timing: (
            [pd.to_numeric(dist, errors='coerce') * 0.000621371 / (pd.to_numeric(time, errors='coerce') / 3600)
             if pd.to_numeric(time, errors='coerce') > 0 else None
             for dist, time in timing.items()],
            [pd.to_numeric(dist, errors='coerce') * 0.000621371 / (pd.to_numeric(time, errors='coerce') / 3600) < 40
             if pd.to_numeric(time, errors='coerce') > 0 else False
             for dist, time in timing.items()]
        )
    )
)
```
### 5. Extract Congestion Details and Coordinates
This step finds the exact time and location of congestion events.
```python
# Extract congestion details (timestamps & distances)
df['congestion_details'] = df.apply(
    lambda row: [
        (convert_to_central_time(row['datetime_utc']), pd.to_numeric(dist, errors='coerce') * 0.000621371)
        for dist, flag in zip(row['road_timing_parsed'].keys(), row['congestion_flags_40mph'])
        if flag and pd.to_numeric(dist, errors='coerce') is not None
    ],
    axis=1
)

# Find coordinates for congestion points
df['congestion_coords'] = df.apply(
    lambda row: get_congestion_coordinates(row['polyline'], [dist for _, dist in row['congestion_details']]),
    axis=1
)
```
### 6. Generate the Congestion Map
The congestion locations are plotted on a Folium map using small red dots.
```python
# Map congestion points
congestion_map = folium.Map(location=[39.048786, -94.484566], zoom_start=12)

# Add small red dots for congestion points
for idx, row in df.iterrows():
    if 'congestion_coords' in row and 'congestion_details' in row:
        for coord, detail in zip(row['congestion_coords'], row['congestion_details']):
            time, distance = detail
            folium.CircleMarker(
                location=coord,
                radius=5,  # Size of the dot
                color="red",
                fill=True,
                fill_color="red",
                fill_opacity=0.8,
                popup=f"<b>Congestion Details</b><br>Distance: {distance:.2f} miles<br>Time: {time}"
            ).add_to(congestion_map)

# Save the map
congestion_map.save("congestion_map_with_dots1.html")
print("Congestion map with red dots saved as 'congestion_map_with_dots1.html'")
```
Expected Output
1. A Folium interactive map visualizing congestion points.
2. Red markers indicate congested areas.

# Step 6: Integrating Congestion, Routes, Cameras, and DMS into a Combined Traffic Map

This step brings together all traffic analysis components into a single interactive map, displaying:
- **Traffic routes** (including alternative routes).
- **Congestion points** (identified based on speed thresholds).
- **CCTV cameras** (blue markers).
- **Dynamic Message Signs (DMS)** (green markers).

###  Generate Interactive Map with Routes, Congestion, and Assets
This script plots:
1. Traffic routes (multiple alternative paths)
2. Congestion points (red markers)
3. CCTV cameras (blue markers)
4. DMS signs (green markers)
```python
# Initialize map
traffic_map = folium.Map(location=[39.048786, -94.484566], zoom_start=12)

# Add congestion points (red dots)
for idx, row in df.iterrows():
    if row['congestion_coords']:
        for coord in row['congestion_coords']:
            folium.CircleMarker(
                location=coord,
                radius=5,
                color="red",
                fill=True,
                fill_color="red",
                fill_opacity=0.8,
                popup=f"<b>Congestion Detected</b>"
            ).add_to(traffic_map)

# Define route colors
route_colors = ["blue", "green", "black", "purple", "orange", "darkred"]

# Plot traffic routes
for idx, row in df.iterrows():
    if row['polyline']:
        for i, route in enumerate(row['polyline']):
            color = route_colors[i % len(route_colors)]
            folium.PolyLine(
                locations=route,
                color=color,
                weight=3,
                opacity=0.8,
                tooltip=f"Route {i + 1} from {row['origin']} to {row['destination']}"
            ).add_to(traffic_map)

# Plot CCTV Cameras
for _, row in camera_df.iterrows():
    folium.Marker(
        location=[row["Latitude"], row["Longitude"]],
        popup=f"Camera: {row['Name']}",
        icon=folium.Icon(color="blue", icon="camera"),
    ).add_to(traffic_map)

# Plot DMS Message Signs
for _, row in dms_df.iterrows():
    folium.Marker(
        location=[row["Latitude"], row["Longitude"]],
        popup=f"DMS: {row['Name']}",
        icon=folium.Icon(color="green", icon="info-sign"),
    ).add_to(traffic_map)

# Save the map
traffic_map.save("traffic_map_with_routes_cameras_congestion.html")
print("Map saved as 'traffic_map_with_routes_cameras_congestion.html'")
```
Expected Output
1. A fully interactive map displaying:
2. Traffic routes (multiple paths)
3. Congestion points (red markers)
4. CCTV Cameras (blue markers)
5. DMS Signs (green markers)

## Conclusion

This Arrowhead Stadium Traffic Analysis provides a detailed geospatial assessment of traffic conditions before and during a major sporting event. The study combines real-time traffic data, congestion analysis, and existing ITS infrastructure to generate actionable insights. By identifying pinch points, traffic flow inefficiencies, and underutilized alternative routes, this analysis helps in planning effective traffic management strategies. The integration of CCTV cameras and DMS locations further enhances the event traffic control system.

This workflow can be adapted and extended for future large-scale events, allowing transportation planners to continuously improve their ITS deployment and congestion mitigation efforts.
