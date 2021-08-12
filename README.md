# flight_delay_prediction

## 1. Project Overview

### 1.1 Business Case:
As both business and leisure travel picks up, airlines are struggling to match increase consumer demand with supply. As a result, the industry expects more and more flight delays that hinder both airlines and passengers. Flight delays are one of the leading cost centers for airlines and one of the largest sources of customer unsatisfaction over the long term. As a whole, flight delays cost airlines and passengers more than $28 billion in 2018 in both direct and indirect costs. There is growing interest for airlines to predict flight delays beforehand in order to optimize operations and improve customer satisfaction. This could also help airline regulators and consumer interest groups better understand the root causes of flight delays and create meausres to better suite the interests of consumers.

### 1.2 Problem Definition:
There is budding interest amongst airline industry executives to more accuratly predict flight delays in order to better understand the root causes of delays, proactively optimize flight operations, and improve customer satisfaction. We have been tasked with predicting flight delays using datasets derrived from both flight delays and weather patterns in the United States.

More specifically, we have been tasked to predict departure delay/no delay, where a delay is defined as 15-minute delay (or greater) with respect to the planned time of departure. This prediction should be done two hours ahead of departure as to give airlines time to act accordingly to help passangers and staff.

### 1.3 Literature Review:
Academics and aircraft carriers alike have studied the subject of aircraft delays and accurate predictionb for years. In fact, the number of papers on the subject increased nearly 90% from 2007 to 2017. Most notably, machine learning has made ample headway into the space, with machine learnning and probablistic modeling accounting for nearly 66% of all research in the subject area.

The industry standard leading accuracy measures for the subject remain around ~85% accuracy and ~87% recall across all recent studies. Notable pieces of liturature achieving these marks include ```Loris Belcastro, Fabrizio Marozzo, et al```. as well as Navoneel Chakrabarty's A Data Mining Approach to Flight Arrival Delay Prediction for American Airlines. A broader literature review on flight delay prediction can be viewed with the work of ```Alice Sternberg, Jorge Soares, et al.```

### 1.4 The Data Pipeline:
We have been given the following data to base our on-time flight performance. This data originated from the ```U.S. Department of Transportation``` and consisted of the following dataset.

**1) Flights Table:** This table tracks significant airline features from all flights arriving and departing from all major US airports for the 2015-2019 timeframe. We were also given the data from Q1 2015 as well as 1H 2015 to test our airline results

**2) Weather Table:** We were also given weather data derrived from the National Oceanic and Atmospheric Administration. This data gives the reader an array of weather features (tempearature, wind speed, weather observations, etc.) from all U.S. weather stations.

**3) Stations Table:** This table contains the geo data of all weather stations and its neighbors.

**4) Airport code Table:** We use the list of airport codes from https://github.com/jpatokal/openflights/tree/master/data. It contains the info of the airports and timezones.

For this project, we are going to rely on the new Delta Lake technology of Databricks creating an ETL pipeline in which we are loading the data from the parquet and csv files then save as delta files. From those delta files, we will create new data tables in which we can easily transform the fields easily with 'update' and 'delete' commands. We can also quickly query the data with SQL for EDA and perform feature engineering for the modeling.

The data extraction work can be seen on this repo ```feature_engineering```.

## 2. Exploratory Data Analysis:
In this section of the project, we analyized the datasets seperately and then together to better understand the data and aid us in coalescing our final join. We also used a weather stations dataset that helped us better understand the locations for our joins.

The notebook can be found in this rep ```eda```

## 3. Data Setup & Join:
**Initial Data Cleansing:**
We performed some data filtering and cleansing before we perform feature enginnering.

**Weather Data:**
1. Excluded non-us weather data
2. Excluded columns where 50% or more records were nulls
3. Removed records with duplicate report types
4. Aggregated records to nearest hour
5. Imputed numerical nulls with 7 day aggregates
6. Imputed categorical nulls with First available values

**Airlines Data:**
1. Excluded ‘Cancelled’ or ‘Diverted’ flights
2. Cleaned Null target variable (dep_del15) where scheduled and actual departure values matched (4,379 total were fixed)
3. Removed remaining records where target variable was still Null
4. Removed unwanted attributes
5. Added necessary time attributes to be used for feature engineering
5. Converted all timestamps to UTC
6. 4 airports (IFP, EAR, XWA, TKI) didn’t have weather station (1% of total flights, ignored)

**Table JOINs for Model Input**
We performed the JOIN using below criteria: We join airlines and weather data using:

1. Join Weather (at BOTH 'Origin' and 'Destination' Airports) with Airline Original on
    a. Airport Code
    b. Date (Flight Date = Weather Report Date) 
    c. Hour (Scheduled Airline Departure - 3 Hours = Weather report Hour)
2. Join Delays By Airport (at BOTH 'Origin' and 'Destination' Airports) table with Airline Original on
    a. Airport Code
    b. Hour (Scheduled Airline Departure - 3 Hours = Delays by airport Hour)
3. Join Delays By Airport By Hour By Airline Carrier table with Airline Original on
    a. Airport Code
    b. Hour (Scheduled Airline Departure - 3 Hours = Delays By Airport By Hour By Airline Carrier Hour)
    
## 4. Feature Engineering:
The notebook can be found in the repo ```feature_engineering```
Feature Engineering for our project consists of 2 parts for 2 datasets below:
1. Weather Data
2. Airlines Data

#### 4.a.: Weather Feature Engineering

We use 'User Defined functions' to parse Weather Data and extract meaningful features as each of the below weather attribute contains information about multiple aspects within:
1. WND - WIND-OBSERVATION
2. CIG - SKY-CONDITION-OBSERVATION
3. VIS - VISIBILITY-OBSERVATION
4. TMP - AIR-TEMPERATURE-OBSERVATION
5. DEW - DEW POINT
6. SLP = Sea Level AIR-PRESSURE-OBSERVATION

For each of these attributes, we performed one or more of these steps:
1. Split column on ','
2. Cast column as integer type
3. Replace missing values, such as '999', with 'None'
4. Drop rows where wind direction quality or wind speed quality is erroneous
5. Drop the original columns after extraction

#### Airlines related features: 
From flights data EDA, we observed that there are two main categories of features that will have the biggest effect on our model:
   
***1. Root Cause Delay:*** Root cause delay is further sub-divided into below categories:

***a. Airport related:*** are those that are caused by external forces such as weather, security delays, and delays caused by air traffic control (such as traffic on the runway). Flights data provides below built-in features about the cause of delays:`taxi_out`, `weather_delay`,`nas_delay`,`security_delay` etc. So we captured the aggregations at the overall airport level. These aggregations are used in the subsequently to normalize the hourly aggregations by the airport. For null records, we replaced those with zero as this makes sense given the aggregations we are performing.
    
***b. Airline related:*** The Airline carrier is responsible for management of operations that include how many flights, routes, etc. aggregated a few features by carrier in addition to the airport and the hour. These `avg_dep_delay`, `avg_carrier_delay` features will potentially capture a cascading affect on delays across the airline at a given airport whereas the `num_flights` (by airline) feature would target the root cause of cascading.

***2. Cascade Delay:*** Cascade delay is a feature which we believe could be critical in determining whether a upcoming flight of the same plane will be delayed or not. If an aircraft is used for 2 - 4 flights per day, and, if one of these earlier flights during the day get delayed for one reason or another, the subsequent flights are ought to be delayed as in-air time is hardly ever sufficient to catch up on the previous delay. To do this we have to be very careful not to have data leakage between our train and test sets. First of all, we need to make sure we are looking back at least two hours as our target variable (`dep_del15`) because thats the minimum requirement for advance delay prediction. There are 2 use cases. Since we need to predict 2 hour in advance, it is possible the previous flight has landed before our prediction time, i.e. Use Case 1 OR not, Use Case 2. We look at 2nd last flight for Use Case 2 and for both use cases, we assign '1' if one of the two previous flight was delayed, '0' otherwise. Also, we dont look back for previous flights if the previous one arrived more than 5 hours before prediction and 7 hours in case of 2nd previous flight. It is because if there is sufficient time lag between previous and current flights for the same aircraft, it may not have significant impact on the delay of current flight as aircraft gets sufficient time to catch up on delays.

## 5.Models and Algorithms Implementation:

The notebook can be found in the repo ```model```

### Model Pipeline

We built model pipeline using spark. After feature selection, we applied ``FeatureHasher`` trick to map features to indices in the features vector. Numerical and categorical attributes were preprocessed and features are extracted, transformed and scaled in the pipeline before feeding it into our model.

**Preprocessing steps includes:**

**1.```StringIndexer```** which  encodes a string or categorical columns of label to  to a column of label indices. The output is fed into ```oneHotEncoder```.

**2.```OneHotEncoder```** maps a column of label indices to a column of binary vectors.

**3.```VectorAssembler```** transforms and combines a given list of columns into a single vector column.

**4.```StandardScaler```** transforms a dataset of vector rows, normalizing each feature to have unit standard deviation and/or zero mean. Two parameters are:

```withStd:``` True by default. Scales the data to unit standard deviation.
```withMean:``` False by default. Centers the data with mean before scaling. It will build a dense output, so take care when applying to sparse input.

After feature Engineering, we identified the following features and categorized them into categorical and numerical variables.

**catVariables** = ['month','day_of_week','sch_dep_hour_UTC','CASCADE_DELAY','distance_group','op_unique_carrier','origin_airport_id','first_WND_wind_obs_origin','first_WND_wind_obs_dest', 
               'first_VIS_var_origin','first_VIS_var_dest','first_CIG_ceil_vis_origin','first_CIG_ceil_vis_dest']

**numVariables** = ["avg_dep_delay_origin","avg_dep_delay_dest", "pct_dep_delayed_origin", "pct_dep_delayed_dest","avg_weather_delay_origin","avg_weather_delay_dest","avg_nas_delay_origin",
                "avg_nas_delay_dest",'WND_dir_angle_avg_origin','WND_dir_angle_avg_dest',"avg_sec_delay_origin","avg_sec_delay_dest","flight_count_carrier","avg_dep_delay_carrier",
                "avg_carrier_delay_carrier","WND_speed_avg_origin","WND_speed_avg_dest","CIG_avg_ceil_ht_origin","CIG_avg_ceil_ht_dest","avg_TMP_air_temp_origin","avg_TMP_air_temp_dest"]
                
These features were used throughout our model experimentations

### 5.1 Baseline model: 
We implemented a baseline model using logistic regression. Though the baseline accuracy is impressive at ```83.5%```, precision showed a considerable average strength in performance. Unfortunately, recall is ```38.1%```. Also observed is the performnace of class ```0``` . It showed SOTA performance. However,class ```1``` results are very low. This is likely due imbalanced dataset where about about 80% of cases are negative thereby assigning negative to majority cases, which led the model to bias towards class ```0``` resulting in low precision and recall for the positive cases.

### 5.2 Baseline model with rebalanced dataset: 
Further investigation showed a disproportionate ratio of observations in each class confirming the imbalance dataset. Without further transformations, the model would result in bias towards 0 prediction favoring Not delayed. There are many approaches to deal with problem, but our team adopted undersampling the majority class without replacement. We also introduced a rebalancing strength (alpha) to down sample the negative class in order to boost the recall. This significantly improved the results. with precision and recall having an improved scores of ```89.8% and 65.3%``` respectively.

### 5.3 Logistic Regression:
Logistic Regression model is our first experiment after our base model. Using parameter grid, we specified some hyperparameters and evaluator metrics using ```metricName="precisionByLabel"``` to further improve the performance of our model. Overall performance showed and improved results with precision of 87.0% and recall of 77.2%. The trade-off between precision and recall was greatly improved as compared to base model with rebalanced dataset

### 5.3 SVM:
We also implemented SVM model. It showed quite an impressive results. It performed even better than the logistic regression model with precision of 87.4% and recall of 84.8%, about 2.6% difference in precision-recall.

### 5.4 Random Forest:
In our Random Forest Model, We redesigned the model pipeline to remove Standard scaler features since this may affect the performance of this essemble model. The result showed the highest precison of 90.1%, but didn't perform so well on recall with 65.8%. SVM still lead with the best precision-recall trade-off.

### 5.4 XGBoost:
We tried XGBoost on our last experiment. The result showed a significant improvement when compared to random forest in term of precision-recall trade-of. It had an Impressive precision of 90.0% and an improved recall of 75.9%. We adopted it as our final model the best performance of number of trees = 20.

### 5.5 Deployment Model:
Our deployment model uses XGBoost with 20 number of trees. We observed precision 88.5% and recall of 75.6%. Please to the notebook for detailed implementation.

## 6. Conclusions

It is quite evident that the ```cascading``` affect of previous flight on the same aircraft was the predominant feature in the model. We anticipated it to be a prominent feature but not with such a dominant margin. Interestingly, a handful of the weather features are present in the top ten features. 

Overall, we tried a variety of models and observed varied time for each. In terms of performance, each models was evaluated using precision as key metric. However, other metrics such as recall, precision-recall, ROC and F1 scores were significantly considered when tunning our hyperparameters. XGBoost was used as our final model, it was trained on 2016 dataset and tested on 2018 dataset with precision of 88.5% and recall of 75.6%. Precision-Recall showed an impressive scores of 90.0% with ROC and F1 of 75.4% and 81.6% respectively. It is worth to note that limited hyperparameter tunning was done on our XGBoost model due to it's computional complexity and cost. 


We produced an effective predictive model given the complexity of the problem, however there is a potential for further exploration, as combination of feature engineering and combination models.

We propose several enhancements that could be made to improve our model:
1. Understand additional weather features especially the ones which were mostly nulls and derive suitable imputation and make those features impactful for delay prediction.
2. Further improvements on the model performance that would include additional hyperparameter tunning.

