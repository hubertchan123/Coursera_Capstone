# IBM Capstone Project: The Battle of Neighborhoods
## Recommending prime housing locations


### 1.1 Introduction

In Singapore, the large majority of land is government-owned. Thus, most of her citizens live in public housing on leased land. The Housing and Development Board (HDB), the government agency overseeing housing development, oversees and builds the majority of new housing; these types of flats are colloquially called “HDB flats” by locals.

The HDB and private developers secure land at auctions and some higher-end housing is hence privately developed. Singapore’s public housing, or HDB flats are commonly sold to middle-income buyers and they have the right to live in their flat until the building’s 99-year lease expires. 

### 1.2 Business Problem

Land is a precious resource in Singapore, naturally, purchasing a home in the country is an important financial commitment. In this report, we will be taking up the hypothetical scenario as a local real estate agency that seeks to help middle-income buyers seek out prime housing locations in Singapore based on their preferences. Thus, the main objective of this project is to provide different housing location options based on different HDB types of residential clusters generated.

### 2. Data

This report will utilise data from the following datasets and APIs,

A. https://data.gov.sg/dataset/resale-flat-prices — Resale Flat Prices on Data.gov.sg 
B. https://docs.onemap.sg/ — OneMap API to obtain the geo-coordinates in Singapore, specifically, planning areas as well as HDB flats in resale dataset
C. https://developer.foursquare.com/docs/places-api/ — Foursquare API will be used to explore common venues surrounding the planning area in Singapore

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
Methodology section which represents the main component of the report where you discuss and describe any exploratory data analysis that you did, any inferential statistical testing that you performed, if any, and what machine learnings were used and why.

### 4. Results
Results section where you discuss the results.

### 5. Discussion
Discussion section where you discuss any observations you noted and any recommendations you can make based on the results.

### 6. Conclusion
Conclusion section where you conclude the report.


### Blogpost
3. Your choice of a presentation or blogpost. (10 marks)
