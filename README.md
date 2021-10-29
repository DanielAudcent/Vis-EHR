# Vis-EHR
MSc Dissertation project - Interactive geospatial visualisation of COVID-related data in the UK

The repository shall be used as a version control during the implementation stage of this MSc Dissertation project.
This project will be submitted in partial fulfillment of the conditions for the award of the degree: MSc Data Science.

## Application URL
Link: https://da-uk-geovis.netlify.app/

## Case study
Link: https://forms.office.com/r/RpkgqB5S9h

## Instructions for use:

#### Zoom + Pan:
The choropleth map has a built-in functionality for the user to zoom in (Double left mouse click or mouse wheel) and out (mouse wheel), as well as panning (drag mouse cursor) which allows the user to make the smaller areas within England, Wales and Scotland more visible. 

#### Tooltip:
By hovering the mouse cursor over an area on the choropleth map, the user can view a summary of the information available on the chosen area such as name, population count, number of cases and case rate. A caret symbol (e.g ▲ or ▼) next to the case count and case percent values represents whether the value is above or below the average value across all area within England, Wales and Scotland, whereas a hyphen-minus (-) indicates that the value is equivalent to the average.

#### Selection options:
This visualisation tool enables the user to select multiple areas through a mouse click, bringing up more detailed information on each area selected in the 'Selection info' panel, whilst also building a chart within the 'Chart comparison' area. This provides the user with an easy way to compare the properties of multiple areas selected through their preferred method. The selection of each area can be undone by clicking the same area again on the map.

The bars on the case rate distribution histogram can also be selected through a mouse click, which will draw a single circle at the centroid of every area contained within the bar selected, making it easier for the user to identify those areas. Again, this action be undone by clicking the same bar the user has selected. Only one bar (case rates group) can be selected at a time.

#### Range slider:
Found at the bottom of the page, the range slider allows the user to filter the dataset according to the users requirements. Areas which fall outside of the user specified range will be shown in a monotone light blue colour, drawing the users attention to the remaining datapoints falling within the range specified. As the user updates the range slider, the case rate histogram automatically updates after a short delay, colouring any bars outside the range white, any bars containing one of the range endpoints a lighter shade of green, whilst the other bars remain a green colour.

#### Colour scheme dropdown:
A dropdown option is available in the 'Options' panel, which allows the user to select a different colour palette for the choropleth map. Several schemes are available including Deut/Prot/Trit friendly options for users with colour vision deficiencies.

#### Chart type selection:
The user has the option to select a bar chart to be shown in the 'Comparison chart' area, whch allows the user to compare each selected area population counts. Alternatively, the user can opt for the radial chart, where all available features within the dataset can be compared, provide more of a profiled outlook for each of the selected areas.

#### Reset button:
This button can be used to restore the visualisation tool back to a default state, clearing any selections the user has made and zooming out the map to the maximum level. Occasionally, the application can run slower than usual which may cause this button to seem ineffective - if this is the case for you, refreshing the webpage using F5 will achieve the same outcome.

## Data Sources used
### Shapefiles

* England (Scraped at MSOA level)
https://coronavirus.data.gov.uk/details/interactive-map/cases

* Scotland (Intermediate zones)
https://data.gov.uk/dataset/133d4983-c57d-4ded-bc59-390c962ea280/intermediate-zone-boundaries-2011

* Wales (MSOA file download):
https://lle.gov.wales/catalogue/item/MiddleLayerSuperOutputAreasMSOA/?lang=en

### MSOA / Intermediate zone codes and names:

* England / Wales (MSOA):
https://geoportal.statistics.gov.uk/datasets/register-of-geographic-codes-june-2021-for-the-united-kingdom/about

* Scotland (Intermediate zone):
https://www.gov.scot/publications/small-area-statistics-reference-materials/

### Population counts

* England / Wales (MSOA) mid-2019 SAPE22DT4 edition :
https://www.ons.gov.uk/peoplepopulationandcommunity/populationandmigration/populationestimates/datasets/middlesuperoutputareamidyearpopulationestimates

* Scotland Intermediate zones Population (all 1279 intermediate zone as of 2011):
https://www.opendata.nhs.scot/dataset/7f010430-6ce1-4813-b25c-f7f335bdc4dc/resource/93df4c88-f74b-4630-abd8-459a19b12f47

### Covid cumulative cases data:

* England:
https://api.coronavirus.data.gov.uk/v2/data?areaType=msoa&metric=newCasesBySpecimenDateRollingSum&format=csv

* Wales (Reports on past 7 days):
7th July 2021 - https://public.tableau.com/app/profile/public.health.wales.health.protection/viz/RapidCOVID-19virology-Public/Headlinesummary

* Scotland:
https://www.opendata.nhs.scot/gl/dataset/covid-19-in-scotland/resource/8906de12-f413-4b3f-95a0-11ed15e61773
