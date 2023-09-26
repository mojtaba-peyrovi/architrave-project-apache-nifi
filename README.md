# Apache Nifi Assessment By Architrave

Author: Mojtaba Peyrovi - Data engineer



Contents:


* Walk through the solution
    - The process overview
    - what was achived and what didn't work
    - How to improve the results

* How to run the project?
    - docker setup
    - GitHub

## 1. Walk through the solution:

__Overview:__ It was my very first time working with Nifi and I enjoyed a lot learning a new platform and was very intersting to know different approached to SCD than I already know. Overal it was a very positive experience and I learned so much, although it can be improved a lot and this can be seen as a Proof of Concept. 


### 1.1 The process overview:
As we can see below, the process starts with a CDC processor to capture any change to our source table, __products_catalog.

Next, we split the flow into two routes based on whether the change in adding a new row, or updating a current row. In an ideal world, we need to be able to handle delete, DDL, commit, and begin events, but I will focus on add and update for this POC.
<img src='./screenshots/Screenshot 2023-09-26 222323.png'>


#### 1.1.1 First route: NEW ROW ADDED
 It belongs to the times when a new record with a unique ProductID is being added to the source table. The process of testing whether the incoming row has a unique ProductID should be handled in the upstream pipeline and no need to test it here. The process is straightforward as we can see below:

<img src='./screenshots/Screenshot 2023-09-26 230156.png'>

These are the steps when a new record comes into the source table:

__A.__ All metadata from the incoming JSON via the FlowFlie get removed and only the key-value pairs of actual data gets stored as a flat JSON.

__B.__ Using JoltTranformJSON processor, we add the following three columns:

```
    - valid_from: It takes the timestamp of now, converted to a MYSQL friendly format
    - valid_until: It takes a null value as the record is not expired yet
    - is_current: takes the values of 'Y' as the latest version of a row in the source table
```
In order to add the above three columns, we need to write specifications as below:

```
[{
  "operation": "default",
  "spec": {
    "*": {
      "valid_from": "${now():toNumber()}"
    }
  }
},

{
  "operation": "default",
  "spec": {
    "*": {
      "valid_until": null
    }
  }
},

{
  "operation": "default",
  "spec": {
    "*": {
      "is_current": "Y"
    }
  }
}]
```


__C.__ Prepare the data for writing on MYSQL and write them into __products_catalog_history__ table.

__D.__ At the end add a log on failure and they info will be stored at:
```
./logs/nifi-app.txt
```
Of course, it is not the best way of logging and in case of a real world data product, we can use processors such as __putEmail__ or find solutions like webhooks or other real time solution to send notifications to users via email, slack, etc. and write custome logs on separate files. 



#### 1.1.2 Second route: AN EXISTING ROW IS UPDATED:

Here is how the flow looks like:

<img src='./screenshots/Screenshot 2023-09-26 234247.png'>


As mentioned in the project description, we are expecting only prices to be changed. In this case, we have two main objectives to be done. 


__TASK 1:__ A new row should be added with the new price and the three SCD-related columns as follows:
```
- valid_from: takes the timestamp of now 
- valid_until: remains null since it is a new record in the target table
- is_current: remains as 'Y'
``` 

To achieve task 1, the following steps are taken:

__A.__ The row with the new price is taken from the source table

__B.__ The above SCD related changes are implemented

__C.__ The JSON inside the FlowFile gets prepared for MYSQL and gets written into the __products_catalog_history__ table. 


__TASK 2:__ The latest row belonging to the same product needs to be found and the following changes to be implemented on its SCD-related columns:
```
- valid_from: remains unchanged.
- valid_until: takes the timestamp of now showing that this record just expired.
- is_current: changes from 'Y' to 'N' showing that this record is not current anymore.
```

In order to implement TASK 2, here are the steps taken:

__A.__ Using lookupRecord, we find the records related to the productID of the updated product. 

- We need to remember, a product can have multiple history changes but there is always ONE record with __is_current__ value as 'Y'. Therefore, inside the lookupService we used, I filtered out the results with 'N' value. Here is how the lookupRecord looks like:

<img src='./screenshots/Screenshot 2023-09-26 233716.png'>

And in the lookup service, instead of passing the product_catalog_history table, I passed a SQL query to keep only 'Y' values to be executed as a subquery. 

```
(SELECT * FROM sample_data.products_catalog_history WHERE Is_current='Y') AS current
```

<img src='./screenshots/Screenshot 2023-09-26 234027.png'>

__B.__ The new price extracted from the incoming data from the source, to be used to update the target table later on.

__C.__ The old row which was lookedup, gets extracted and cleaned since the result was not flat, to be edited and update the target table. 


__D.__ Using evaluateJSONPath, we extracted the two SCD values that we want to change (valid_until, is_current) as new variables to be addressed in the next process.

<img src='./screenshots/Screenshot 2023-09-26 234946.png'>


__E.__ Using JoltTransformJSON, we update the two SCD columns, as shown in the following jolt specification:

```
[

{
   "operation": "modify-overwrite-beta",
   "spec": {
     "valid_until": "${now():format('yyyy-MM-dd hh:mm:ss')}"
  }	
}, 

{
   "operation": "modify-overwrite-beta",
   "spec": {
     "Is_current": "N"
  }	
}

]
```

__F.__ The data gets written to the history table.


### 1.2 what was achived and what didn't work?
As mentioned above, this project is far from perfect according to the industry standards because of me being absolutely new to the tool, the learning curve of Nifi and lack of a strong community around Nifi to help with debugging and learning.

__Achievements:__ 
- The core functionality of SCD type 2 is implemented as expected
- Some basic logging added as an example of data quality checking
- Docker implementation with some limitations

__What is not achieved?__

- The update flow does not work with more than one update, meaning that if a record gets updated multiple times, the old prices get replaced witht the latest history. There must be a step right before we update the latest history -- this step is called 'update the required fields' as per the following screenshot-- to overweite only the history with is_current='Y'. Unfortunately, I didn't find a way to handle this.  

<img src='./screenshots/Screenshot 2023-09-27 000703.png'>

- Also, Task 2 of the second route should be running before Task 1. In order to do so, I tried to implement a delay. My research showed two ways: 

    - Using Wait processor
    - Changing the run schedule of the first processor in Task 1 to be delayed 

Somehow, I didn't manage to find the solution due to the short time. 


### 1.3 What can be added for the next iteration?
- Using group processors can increase the readability of the project
- A strong logging system coupled with a powerful notification system can help data quality improvements
- Adding db credentials as environment variables or a config file.
- 