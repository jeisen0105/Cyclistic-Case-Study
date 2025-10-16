# Case Study: Cyclistic Bike-Share Analysis (Google Data Analytics Capstone)

## Introduction

This project presents an analysis of the Cyclistic Bike-Share case study, part of the Google Data Analytics Professional Certificate capstone. The goal is to address key business questions using the six-step data analysis process: Ask, Prepare, Process, Analyze, Share, and Act. 

## Background

Cyclistic is a bike-share program based in Chicago, featuring 5,824 bicycles and 692 docking stations across the city. Unlike other bike-share companies, Cyclistic offers a range of bicycles, including those designed for people with disabilities—such as reclining bikes, hand tricycles, and cargo bikes—making the service more inclusive and accessible.

Cyclistic also provides flexible pricing options: single-ride passes, full-day passes, and annual memberships, allowing it to appeal to a broad range of users. However, the company’s marketing director believes future success lies in increasing the number of annual memberships.

## Scenario

In this hypothetical scenario, I am a junior data analyst on Cyclistic’s marketing analytics team. I will present my findings and recommendations to key stakeholders, including Lily Moreno (Marketing Director) and the Cyclistic executive team.
 
## Step 1: Ask

The business task at hand is to understand the ways annual members and casual riders use Cyclistic bikes differently. By understanding how each type of rider uses Cyclistic we will be able to determine patterns of usage over time which can then be used to designed targeted marketing strategies to convert as many casual riders as possible into annual members. 

## Step 2: Prepare

### Does the Data ROCCC?

When approaching the prepare section it's critical to ensure data is reliable, original, comprehensive, current and cited. The data being used comes from Cyclistic's divvy trip data pertaining to both 2019 Q1 and 2020 Q1 and has been made available by Motivate International Inc. under their [Data License Agreement](https://divvybikes.com/data-license-agreement). Because R was used for analysis, only Q1 data was used due to size and format limitations.

The data is public and can be used to investigate how different customer types are using Cyclistic bikes. In order to comply with data privacy issues the data excludes riders personally identifiable information including names or financial details. 

In order to determine whether or not the data source is reliable, original, comprehensive, current and cited I will follow the ROCCC framework. I know the data is reliable and original because it contains accurate, complete and unbiased information on Cyclistic's historical bike trips which come from a primary source. The data also contains all information needed to understand the different ways annual and casual riders use Cyclistic bikes making it quite comprehensive. Additionally because the data sources are provided publicly by Cyclistic it can be referenced easily. Finally even though our data is 5 to 6 years old it is still young enough for this behavioral analysis. 

### Preparing RStudio

I first installed the necessary packages tidyverse and conficted and used the conflict_prefer command to ensure consistency.

```r
#Install necessary packages
install.packages("tidyverse")
install.packages("conflicted")
library(tidyverse) 
library(conflicted)

#Set dplyr::filter and dplyr::lag as the default choices
conflict_prefer("filter", "dplyr")
conflict_prefer("lag", "dplyr")
```

I then used the read_csv() function to create dataframes for both years of data.

```r
q1_2019 <- read_csv("Divvy_Trips_2019_Q1 - Divvy_Trips_2019_Q1.csv")
q1_2020 <- read_csv("Divvy_Trips_2020_Q1 - Divvy_Trips_2020_Q1.csv")
```

## Step 3: Process

The processing stage involves cleaning and transforming the data to ensure accuracy, consistency, and readiness for analysis. This includes removing incomplete or innaccurate entries and aligning column names/data types across datasets.

#### Renaming columns
The column names in the 2019 data were renamed to match the 2020 schema, and the ride_id and rideable_type columns were explicitly converted to the character data type to allow for correct stacking  

```r
#Rename columns to make them consistent with q1_2020
(q1_2019 <- rename(q1_2019
                   ,ride_id = trip_id
                   ,rideable_type = bikeid
                   ,started_at = start_time
                   ,ended_at = end_time
                   ,start_station_name = from_station_name
                   ,start_station_id = from_station_id
                   ,end_station_name = to_station_name
                   ,end_station_id = to_station_id
                   ,member_casual = usertype))

#Convert ride_id and rideable_type to character so that they can stack correctly
q1_2019 <-  mutate(q1_2019, ride_id = as.character(ride_id)
                   ,rideable_type = as.character(rideable_type))
```

### Combine Datasets

The cleaned quarterly data frames were combined into a single master data frame, all_trips.

```r
#Stack individual quarter's data frames into one big data frame
all_trips <- bind_rows(q1_2019, q1_2020)
```

### Remove Unwanted Columns

Unnecessary columns (like birthyear, gender, and specific station coordinates) that were dropped from the public data in 2020 were removed for consistency.

```r
#Remove non essential columns
all_trips <- all_trips %>%  
  select(-c(start_lat, start_lng, end_lat, end_lng, birthyear, gender, tripduration))
```
### Standardize Data

The 2019 user type labels ("Subscriber" and "Customer") were standardized to match the 2020 labels ("member" and "casual") using the recode() function. 

```r
#Standardize user type labels
all_trips <-  all_trips %>% 
  mutate(member_casual = recode(member_casual,"Subscriber" = "member","Customer" = "casual"))
```

### Add columns

New columns were created to extract granular time data (month, day, year, day-of-week) from the started_at timestamp, and the ride_length column was calculated by finding the difference between the end and start times. The data type for the ride_length column is also converted to numeric.

```r
#Add columns that list the date, month, day, and year of each ride
all_trips$date <- as.Date(all_trips$started_at)
all_trips$month <- format(as.Date(all_trips$date), "%m")
all_trips$day <- format(as.Date(all_trips$date), "%d")
all_trips$year <- format(as.Date(all_trips$date), "%Y")
all_trips$day_of_week <- format(as.Date(all_trips$date), "%A")

#Add a "ride_length" calculation to all_trips
all_trips$ride_length <- difftime(all_trips$ended_at,all_trips$started_at)

#Convert "ride_length" from Factor to numeric 
all_trips$ride_length <- as.numeric(as.character(all_trips$ride_length))
```

### Cleaning combined dataset

After noticing the presence of negative values in the ride_length column and invalid entries in the start_station_name, a new dataframe (all_trips_v2) is created to filter out these non-usable entries. 

```r
#Create a new dataframe (v2) to filter out invalid entries
#Remove rides with negative duration and the 'HQ QR' station, which is known to be a test/placeholder station
all_trips_v2 <- all_trips[!(all_trips$start_station_name == "HQ QR" | all_trips$ride_length<0),]
```

## Step 4: Analyze


