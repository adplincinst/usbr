
# Problem

The Western Water Data Hub [search interface](https://doimspp.sharepoint.com/:b:/r/sites/bor-lincoln-wswc-interop-water-data-hub/Shared%20Documents/General/Visualization%20Tool/National%20Data%20Explorer%20update_10_15_2024.pdf?csf=1&web=1&e=BHU9JD) is designed to search across supported data sources using the EDR API. The proposed search categories and filters are outlined below

# Hub Search Filters


> Note: The following sections make use of the URI schema `jq:` which is intended to show the reader the intended JSON path the the referenced element value within a JSON-LD document if the [jq](https://jqlang.org/) utility was used.  For instance `jq:.geometry.coordinates` references a JSON(-LD) document with the nested property geometry -> coordinates like `{ "geometry":{"coordinates":  <some value>}}` where `<some value>` is the referenced value returned from the `jq` filter `.geometry.coordinates`.

## Filter by Location

Location filters are segmented into (HUC02) Watersheds, States, and Counties. 

> Should we have Reclamation Regions as well?

### Filter by Watershed

Watershed contains a list of HUC02 watersheds that intersect with one or more of the 17 western states within the Bureau of Reclamation (Reclamation) jusrisdiction. Names and  geographic boundaries for these watersheds can be found in the Geoconnex reference feature server [here](https://reference.geoconnex.us/collections/hu02) as  collection item listings by fetching  each collection item resource as a JSON-LD document  `?f=jsonld` and extracting the name `jq:.name` and  (MultiPolydon) coordinates for the watershed  `jq:.geometry.coordinates`.

> Note: need a way to filter all watersheds to only the ones that intersect with the Reclamation regions?


### States

List of [17 western states](https://www.usbr.gov/main/images/region-map-small.jpg) which are under Reclamation's jurisdiction. Geographic boundaries can be found in the Geoconnex reference feature server [here](https://reference.geoconnex.us/collections/states/). Iterating each collection item and fetching its JSON-LD representation `?f=jsonld` will provide the state code `jq:.stusps`, state name `jq:.name`, and (MultiPolygon) coordinates `jq:geometry.coordinates`. 

## Filter by Data Category

### Category

I believe `Category` as shown in the search interface is synonymous with USBR RISE system's `Parameter Group` term. In this context, a canonical Parameter Group designation **COULD BE** be fetched from the JSONLD representation of the `/collections` resource similar to each collection item having a `parameter_names` property in the JSON document representation of the collection item. This collection level list of the canonical  parameter names would be the Data Category/Parameter Group variables that are sourced from the USBR ontology. 

> Need to investigate the feasibility of exposing the parameter group names in `/collections` json(-ld) resource to be able to populate this list and still remain conformant to EDR (I hope)

> Should we change the terminology here? `Parameter Group` is what is familiar to RISE users for instance. 


### Dataset

I believe `Dataset` here is synonymous with USBR RISE system's `Parameter` term. The population of this listing is dependent upon the selection of a `Category` (Parameter Group) filter where parameters are Objects of SKOS `narrower`

## Filter by Provider 


### Provider 

Provider is thought to be the publishing organization of the dataset. This listing could be populated by iterating the EDR `/collections` item links as JSON-LD document resources  `?f=jsonld` where the provider name is  `jq:.provider.name`. The filtering process would be filtering collection items by the user selected provider names. 

> Note: currently all the EDR collection item JSON-LD resource documents  set the provider organization to the `Internet of Water`. This should be changed to the organization that publishes the data (e.g. USGS, Reclamation, etc...) and its related metadata. 



## Time 

Time is the number of years (from present day) that have a period of record.

The JSON-LD representation for each collection item allow for the specification of a period of record `jq:.dataset.temporalCoverage` (see [schema.org here](https://schema.org/temporalCoverage) for details on temporalCoverage format). 

> Currently the value `jq:.dataset.temporalCoverage` is set to null for all collection items. Need to be able to determine the period of record from the source and set this value accordingly so that it can be used by clients to filter by period of record. 

> What isn't clear here is what constitutes a valid period year. For instance, if one time series data point exists for a year is does that count as a year within the period of record? 

# Sequence Diagrams

## Load Location Filters

Describes interactions when Hub Search Page loads

```mermaid
sequenceDiagram
autonumber
actor HubUser
participant HubSearchPage
#participant EDRAPI
participant RefFeatureServer
participant ArcGIS


HubUser ->> HubSearchPage: load
HubSearchPage ->> HubSearchPage: Populate Basin Dropdown
activate HubSearchPage
HubSearchPage ->> RefFeatureServer: GET /collections/hu02
loop For Each Collection ID
HubSearchPage ->> RefFeatureServer: /collections/hu02/items/{collectionId}?f=json
RefFeatureServer -->> HubSearchPage: Feature object
activate HubSearchPage
HubSearchPage ->> HubSearchPage: jq:.properties.name
HubSearchPage ->> HubSearchPage: jq:.geometry.coordinates
deactivate HubSearchPage

end
deactivate HubSearchPage


HubSearchPage ->> HubSearchPage: Populate States Dropdown
activate HubSearchPage
HubSearchPage ->> RefFeatureServer: GET /collections/states
loop For Each Collection ID
  HubSearchPage ->> RefFeatureServer: /collections/states/items/{collectionId}?f=json
  RefFeatureServer-->> HubSearchPage: Feature object
  activate HubSearchPage
  HubSearchPage -->> HubSearchPage: jq:.properties.name
  HubSearchPage -->> HubSearchPage: jq:.geometry.coordinates
  deactivate HubSearchPage
end
deactivate HubSearchPage


HubSearchPage ->> HubSearchPage: Populate USBR Regions Dropdown
activate HubSearchPage
HubSearchPage ->> ArcGIS: GET https://services1.arcgis.com/fBc8EJBxQRMcHlei/arcgis/rest/services/DOI_Unified_Regions/FeatureServer/0
ArcGIS -->> HubSearchPage: GeoJSON

loop For Each Reclamation Region
  HubSearchPage ->> HubSearchPage: Extract Region Coordinates
  HubSearchPage ->> HubSearchPage: Extract Region Name

end
deactivate HubSearchPage

```
## Select  Filters
Describes behavior when user selects filters.

```mermaid
sequenceDiagram
#autonumber
actor HubUser
participant HubSearchPage
participant EDRAPI
#participant RefFeatureServer

critical Select Provider
  HubUser->>HubSearchPage: 
  HubSearchPage->>HubSearchPage: Save filter [provider filter] for EDR Collection items

  option Select Location Filter
  alt 
    HubUser->>HubSearchPage: Select Basin
  else 
    HubUser->>HubSearchPage: Select USBR Region
  else 
    HubUser->>HubSearchPage: Select State
  end
  HubSearchPage-->>HubSearchPage: Save as [location filter]
  
  option Select Data Category
    HubUser->>HubSearchPage: Select Category
    HubSearchPage->>HubSearchPage: Save as [category filter]

  option Select Dataset
    HubUser->>HubSearchPage: Select Dataset
    HubSearchPage->>HubSearchPage: Save as client-side  filter [dataset filter] 

end

loop for each collectionId in EDR collections
  HubSearchPage->>EDRAPI: GET /collections/{collectionId}?f=jsonld
  EDRAPI-->>HubSearchPage: schema:DataCatalog object
  critical schema:DataCatalog provider object equials [provider filter]
    HubSearchPage->EDRAPI: GET /collections/{collectionId}/locations
    HubSearchPage->HubSearchPage: Add features to Map
  end   
end

HubUser->>HubSearchPage: click feature on map
activate HubSearchPage
HubSearchPage->>EDRAPI: Get timeseries for clicked feature
EDRAPI-->>HubSearchPage: timeseries data
HubSearchPage->>HubSearchPage: display timeseries 
opt Change Date Range?
  HubUser->>HubSearchPage: Change date range  
end
opt Download Timeseries as CSV?
  HubUser->>HubSearchPage: Click download
  HubSearchPage-->>HubUser: CSV file
end
deactivate HubSearchPage

```


## Download All Sites (Timeseries Data) 

> NOTE: Currently this thought to be an intensive high-latency operation. This is on HOLD as of 08/24/25


```mermaid
sequenceDiagram
#autonumber
actor HubUser
participant HubSearchPage
participant EDRAPI
#participant RefFeatureServer

HubUser->>HubSearchPage: Download

critical [category filter] is Saved AND 
  
  HubSearchPage->>EDRAPI: GET /collections?parameter-name=[category filter]&f=jsonld
  loop For Each {collectionId} ...
      EDRAPI-->>HubSearchPage: schema:DataCatalog 
      alt [provider filter] is Saved 
           HubSearchPage->>HubSearchPage: filter by [provider filter]
      end
      alt [dataset filter] is Saved
           HubSearchPage->>HubSearchPage: filter by [dataset filter]
  
      end

      alt [location filter] is Saved
           HubSearchPage->>HubSearchPage: set query param 'coords' to [location filter]
           
      end
      alt [last N years] is Saved
           HubSearchPage->>HubSearchPage: set query param 'datetime' to [last N years filter]
     
      end
      HubSearchPage ->> EDRAPI: Get timeseries for {collectionId} using set query params 
      EDRAPI-->>HubSearchPage: Timeseries
      HubSearchPage-->>HubUser: Timeseries

  end
  
end

```

