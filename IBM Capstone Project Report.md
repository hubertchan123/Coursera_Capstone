# IBM Capstone Project: The Battle of Neighborhoods
## Recommending prime housing locations


### 1.1 Introduction

In Singapore, the large majority of land is government-owned. Thus, most of her citizens live in public housing on leased land. The Housing and Development Board (HDB), the government agency overseeing housing development, oversees and builds the majority of new housing; these types of flats are colloquially called “HDB flats” by locals.

The HDB and private developers secure land at auctions and some higher-end housing is hence privately developed. Singapore’s public housing, or HDB flats are commonly sold to middle-income buyers and they have the right to live in their flat until the building’s 99-year lease expires. 

### 1.2 Business Problem

Land is a precious resource in Singapore, naturally, purchasing a home in the country is an important financial commitment. In this report, we will be taking up the hypothetical scenario as a local real estate agency that seeks to help middle-income buyers seek out prime housing locations in Singapore based on their preferences. Thus, the main objective of this project is to provide different housing location options based on different HDB types of residential clusters generated.

### 2. Data

This report will utilise data from the following datasets and APIs,

- A. Resale Flat Prices on Data.gov.sg — https://data.gov.sg/dataset/resale-flat-prices 
- B. OneMap API to obtain the geo-coordinates in Singapore, specifically, planning areas as well as HDB flats in resale dataset — https://docs.onemap.sg/
- C. Foursquare API will be used to explore common venues surrounding the planning area in Singapore — https://developer.foursquare.com/docs/places-api/ 

  #### A. Preparing Resale Flat Prices on Data.gov.sg
  ```
  url = 'https://data.gov.sg/api/action/datastore_search?resource_id=42ff9cfe-abe5-4b54-beda-c88f9bb438ee&limit=103087'
  results = requests.get(url).json()
  # Create Dataframe to store results that we will be using
  hdb_resale_results = results['result']['records']
  hdb_resale = json_normalize(hdb_resale_results)

  #We will be using Town names, addresses, resale prices, size of flat, and remaining leases
  filtered_columns = ['town', 'street_name', 'block', 'resale_price', 'floor_area_sqm', 'remaining_lease']
  hdb_resale =hdb_resale.loc[:, filtered_columns]
  ```
  ![image](https://user-images.githubusercontent.com/49154571/126875913-c80b8b7d-5f5c-4034-ae49-dccd6356a3f3.png)
  
  #### B. Preparing OneMap API to obtain the geo-coordinates
  ```
  # List of addresses
  list_of_address = hdb_east['address'].tolist()
  add_lat = []
  add_long = []
  
  # Obtaining coordinates for 500 addresses

  for i in range(0, len(list_of_address)):
    query_address = list_of_address[i]
    query_string = 'https://developers.onemap.sg/commonapi/search?searchVal='+str(query_address)+'&returnGeom=Y&getAddrDetails=Y'
    resp = requests.get(query_string)

    data_add=json.loads(resp.content)

    if data_add['found'] != 0:
        add_lat.append(data_add["results"][0]["LATITUDE"])
        add_long.append(data_add["results"][0]["LONGITUDE"])

    else:
        add_lat.append('NotFound')
        add_long.append('NotFound')

  # Store this information in a dataframe

  hdb_east['Latitude'] = add_lat
  hdb_east['Longitude'] = add_long
  hdb_east.head(3)
  ```
  ![image](https://user-images.githubusercontent.com/49154571/126875968-c56b6e2a-c926-4c94-8f48-8a64d1b81da9.png)

  #### C. Foursquare API to explore common venues
  ```
  CLIENT_ID = '' # your Foursquare ID
  CLIENT_SECRET = '' # your Foursquare Secret
  ACCESS_TOKEN = '' # your FourSquare Access Token
  VERSION = '20180604'
  LIMIT = 20
  
  def getNearbyVenues(names, latitudes, longitudes, radius=150):

    venues_list=[]
    for name, lat, lng in zip(names, latitudes, longitudes):

        # create the API request URL
        url = 'https://api.foursquare.com/v2/venues/explore?&client_id={}&client_secret={}&v={}&ll={},{}&radius={}&limit={}'.format(
            CLIENT_ID, 
            CLIENT_SECRET, 
            VERSION, 
            lat, 
            lng, 
            radius, 
            LIMIT)

        # make the GET request
        results = requests.get(url).json()["response"]['groups'][0]['items']

        # return only relevant information for each nearby venue
        venues_list.append([(
            name, 
            lat, 
            lng, 
            v['venue']['name'], 
            v['venue']['location']['lat'], 
            v['venue']['location']['lng'],  
            v['venue']['categories'][0]['name']) for v in results])

    nearby_venues = pd.DataFrame([item for venue_list in venues_list for item in venue_list])
    nearby_venues.columns = ['Neighborhood', 
                  'Neighborhood Latitude', 
                  'Neighborhood Longitude', 
                  'Venue', 
                  'Venue Latitude', 
                  'Venue Longitude', 
                  'Venue Category']

    return(nearby_venues)
  
  # Get top 20 venues for every HDB address in a 150m radius

  hdb_venues = getNearbyVenues(names=hdb_east['Neighborhood'],
                                   latitudes=hdb_east['Latitude'],
                                   longitudes=hdb_east['Longitude']
                                  )
  ```
  ![image](https://user-images.githubusercontent.com/49154571/126875997-96f1cab7-2afd-49a9-944c-b8bfde532945.png)


### 3. Methodology
Here is an overview of the methodology / data manipulations conducted
- A. One hot encoding
- B. Get Top Venues
- C. K-means clustering
- D. Elbow Method to determine optimal K-Means
- E. Charting on clusters on map

  #### A. One hot encoding
  ```
  # one hot encoding
  hdb_venues_onehot = pd.get_dummies(hdb_venues[['Venue Category']], prefix="", prefix_sep="")

  # add neighborhood column back to dataframe
  hdb_venues_onehot['Neighborhood'] = hdb_venues['Neighborhood'] 

  # move neighborhood column to the first column
  fixed_columns = [hdb_venues_onehot.columns[-1]] + list(hdb_venues_onehot.columns[:-1])
  hdb_venues_onehot = hdb_venues_onehot[fixed_columns]
  
  hdb_venues_grouped = hdb_venues_onehot.groupby('Neighborhood').mean().reset_index()
  hdb_venues_grouped.head()
  ```
  ![image](https://user-images.githubusercontent.com/49154571/126876295-c4daaedd-cb78-4d98-8b1f-20eb875a4987.png)
  
  #### B. Get Top Venues
  ```
  def return_most_common_venues(row, num_top_venues):
    row_categories = row.iloc[1:]
    row_categories_sorted = row_categories.sort_values(ascending=False)

    return row_categories_sorted.index.values[0:num_top_venues]
    
  num_top_venues = 5

  indicators = ['st', 'nd', 'rd']

  # create columns according to number of top venues
  columns = ['Neighborhood']
  for ind in np.arange(num_top_venues):
    try:
        columns.append('{}{} Most Common Venue'.format(ind+1, indicators[ind]))
    except:
        columns.append('{}th Most Common Venue'.format(ind+1))

  # create a new dataframe
  neighborhoods_venues_sorted = pd.DataFrame(columns=columns)
  neighborhoods_venues_sorted['Neighborhood'] = hdb_venues_grouped['Neighborhood']

  for ind in np.arange(hdb_venues_grouped.shape[0]):
    neighborhoods_venues_sorted.iloc[ind, 1:] = return_most_common_venues(hdb_venues_grouped.iloc[ind, :], num_top_venues)

  neighborhoods_venues_sorted.head()
  ```
  ![image](https://user-images.githubusercontent.com/49154571/126876304-4934ee80-2550-455c-8dc4-4daccf25bba2.png)
  
  #### C. Elbow Method to determine optimal K-Means
  ```
  #Check for optimal cluster size using the elbow method
  sse = {}
  for k in range(1,10):
    kmeans = KMeans(n_clusters=k,random_state=0)
    kmeans.fit(hdb_grouped_clustering_set)
    hdb_grouped_clustering['Cluster Labels'] = kmeans.labels_
    sse[k] = kmeans.inertia_

  import matplotlib.pyplot as plt
  plt.figure()
  plt.plot(list(sse.keys()), list(sse.values()))
  plt.xlabel("Number of clusters")
  plt.ylabel("SSE")
  plt.show()
  ```
  ![image](https://user-images.githubusercontent.com/49154571/126876369-fc7b5238-50ff-4e1e-a190-e651f46628a4.png)
  
  #### D. K-means clustering
  ```
  hdb_grouped_clustering = pd.merge(hdb_grouped_clustering, hdb_venues_grouped, on="Neighborhood")

  # set number of clusters
  kclusters = 5

  hdb_grouped_clustering_set = hdb_grouped_clustering.drop('Neighborhood', 1)
  hdb_grouped_clustering.drop(['Cluster Labels'], axis=1, inplace=True)

  # run k-means clustering
  kmeans = KMeans(n_clusters=kclusters, random_state=0).fit(hdb_grouped_clustering_set)

  # check cluster labels generated for each row in the dataframe
  kmeans.labels_[0:10]
  
  # add clustering labels
  hdb_grouped_clustering.insert(0, 'Cluster Labels', kmeans.labels_)
  hdb_grouped_clustering.head(1)
  ```
  ![image](https://user-images.githubusercontent.com/49154571/126876325-be889da7-c704-4c94-bbd2-f0f022fcfbdf.png)
  
  #### E. Charting on clusters on map
  ```
  address = 'Singapore, SG'

  geolocator = Nominatim(user_agent="sg_explorer")
  location = geolocator.geocode(address)
  latitude = location.latitude
  longitude = location.longitude
  print('The geograpical coordinate of Singapore are {}, {}.'.format(latitude, longitude))
  
  # create map
  map_clusters = folium.Map(location=[latitude, longitude], zoom_start=11)

  # set color scheme for the clusters
  x = np.arange(kclusters)
  ys = [i + x + (i*x)**2 for i in range(kclusters)]
  colors_array = cm.rainbow(np.linspace(0, 1, len(ys)))
  rainbow = [colors.rgb2hex(i) for i in colors_array]

  # add markers to the map
  markers_colors = []
  for lat, lon, poi, cluster in zip(hdb_merged['Latitude'].astype(float).values.tolist(), hdb_merged['Longitude'].astype(float).values.tolist(), hdb_merged['Neighborhood'], 
                                  hdb_merged['Cluster Labels'].astype(int)):
    label = folium.Popup(str(poi) + ' Cluster ' + str(cluster), parse_html=True)
    folium.CircleMarker(
        [lat, lon],
        radius=5,
        popup=label,
        color=rainbow[cluster-1],
        fill=True,
        fill_color=rainbow[cluster-1],
        fill_opacity=0.7).add_to(map_clusters)

  map_clusters
  ```
  ![image](https://user-images.githubusercontent.com/49154571/126877306-d5cec5bb-2dcc-4d48-b680-212d06d35e89.png)
  ![image](https://user-images.githubusercontent.com/49154571/126877321-5885c0c6-a554-455b-9928-3ee7612bdbab.png)
  ![image](https://user-images.githubusercontent.com/49154571/126877317-8a7cfb13-9fa1-4610-8e2a-55935228b80a.png)
  ![image](https://user-images.githubusercontent.com/49154571/126877335-78165053-f98e-4978-b4d7-1672edb1aa19.png)
  ![image](https://user-images.githubusercontent.com/49154571/126877342-f71432bd-e6c6-41ec-a30e-a0f0b62ccf8d.png)
  ![image](https://user-images.githubusercontent.com/49154571/126877351-81ba5c95-8f2e-4394-a1ab-bb29e3d06294.png)
  
  Red: Cluster 1
  Purple: Cluster 2
  Blue: Cluster 3
  Green: Cluster 4
  Orange: Cluster 5

### 4. Results & Discussion

#### Cluster 1: Affordable & Active
```
       Cluster Labels   Resale Price  Floor Area
count            88.0      88.000000   88.000000
mean              0.0  418183.954545   98.943182
std               0.0   55907.405046   13.345418
min               0.0  255000.000000   67.000000
25%               0.0  396500.000000   92.000000
50%               0.0  424000.000000  104.000000
75%               0.0  450000.000000  104.000000
max               0.0  565000.000000  128.000000
```
![image](https://user-images.githubusercontent.com/49154571/126876692-36dcfdc2-0964-49fe-9921-b22e7ade6667.png)

- On average, flats in this cluster costs $418,183 and is 99 sqm floor area
- This puts it at $4,224 per sqm, making this cluster the cheapest cluster of flats
- This cluster of flats seems to have more sports & recreational venues nearby.
- As a result, we can recommend this cluster of flats to buyers who are looking for affordable flats and prefer to have an active lifestyle.

#### Cluster 2: Spacious & Accessible
```
       Cluster Labels   Resale Price  Floor Area
count           171.0     171.000000  171.000000
mean              1.0  573557.467836  128.549708
std               0.0  116452.868010   17.951295
min               1.0  300000.000000   67.000000
25%               1.0  491500.000000  120.000000
50%               1.0  552000.000000  127.000000
75%               1.0  640000.000000  146.000000
max               1.0  915000.000000  156.000000
```
![image](https://user-images.githubusercontent.com/49154571/126876811-92e32d33-cc17-4acb-b831-70b7a969d182.png)

- On average, flats in this cluster costs $573,557 and is 129 sqm floor area (largest set of flats)
- This puts it at $4,446 per sqm
- This cluster of flats seems to offer good transportation options as well as food close by.
- As a result, we can recommend this cluster of flats to buyers who are looking for bigger flats and prefer to have ready access to public transport.

#### Cluster 3: Compact & For Foodies
```
       Cluster Labels   Resale Price  Floor Area
count           190.0     190.000000  190.000000
mean              2.0  313637.705263   73.105263
std               0.0   54950.550580   11.614144
min               2.0  240000.000000   59.000000
25%               2.0  280000.000000   67.000000
50%               2.0  300000.000000   68.000000
75%               2.0  340000.000000   82.000000
max               2.0  603000.000000  134.000000
```
![image](https://user-images.githubusercontent.com/49154571/126876843-8907eb5c-305e-44fe-ad7d-e3c43c552588.png)

- On average, flats in this cluster costs $313,637 and is 73 sqm floor area (smallest set of flats)
- This puts it at $4,296 per sqm
- This cluster of flats has lots of food options close by.
- As a result, we can recommend this cluster of flats to buyers who are looking for smaller flats (or living in smaller numbers) and prefer to have ready access to food.

#### Cluster 4: Nature & Recreation
```
       Cluster Labels   Resale Price  Floor Area
count           117.0     117.000000  117.000000
mean              3.0  533238.145299   98.777778
std               0.0  115975.543921   18.163107
min               3.0  258000.000000   47.000000
25%               3.0  460000.000000   93.000000
50%               3.0  500000.000000   97.000000
75%               3.0  585000.000000  108.000000
max               3.0  838000.000000  145.000000
```
![image](https://user-images.githubusercontent.com/49154571/126876877-53b079f8-4555-44dd-b0e7-407eb37792be.png)

- On average, flats in this cluster costs $533,238 and is 99 sqm floor area
- This puts it at $5,386 per sqm, making this cluster the most expensive cluster of flats
- This cluster of flats seems to be close to nature and has a good mixture of sports and recreational venues
- As a result, we can recommend this cluster of flats to buyers who are looking for flats that are close to nature and seeking an active lifestyle.

#### Cluster 5: Events & Recreation
```
       Cluster Labels   Resale Price  Floor Area
count            65.0      65.000000   65.000000
mean              4.0  452381.169231  106.569231
std               0.0  100588.643528   23.075941
min               4.0  275000.000000   67.000000
25%               4.0  374000.000000   92.000000
50%               4.0  442000.000000  105.000000
75%               4.0  513000.000000  124.000000
max               4.0  750000.000000  157.000000
```
![image](https://user-images.githubusercontent.com/49154571/126876897-a7d127cf-b486-4a29-b4c9-31c26861faa6.png)

- On average, flats in this cluster costs $452,381 and is 107 sqm floor area
- This puts it at $4,227 per sqm
- This cluster of flats seems to be close to event spaces and sports & recreational venues
- As a result, we can recommend this cluster of flats to buyers who are looking for flats that are close to event venues and recreation.


### 6. Conclusion
In conclusion, this report was able to generate clusters of housing flats in the East region of Singapore with distinct characteristics to help the hypothetical real estate agency find suitable housing locations for its clients.

### Blogpost
TBC
