# Description

This is an article taht I wrote in **2015** regarding how to unitizing the new features of SAP HANA.

# Background:
E-commerce is becoming a trend and getting more and more popular in retail business. However, certain products, like shoes or clothes, still need the involvement of customer’s personal experience. Such generates a brand-new business concept called O2O, in terms of online-to-offline, which is a strategy to draw customers from the online channels to the physical stores. And, to access the online information, there are various approaches. Nowadays, the most popular and efficient way is through smart phone. In this article, I will demonstrate an application that achieves these two business models. 

# Business Case:
The O2O concept and mobile Apps can be adopted in many requirements of today’s business. For instance, in all walks of business, running campaign is the common way to promote products. Today, one of the most efficient, or cheapest, methods to launch a campaign is through social media, email or SMS. Let’s suppose a fashion footwear company wants to put a new product into market. It publishes an advertisement on the social media, sending the message to the targeted customers. People who see the information in their phone may want to go to the store and pick a fitted pair immediately. Usually, they prefer to go to those stores that have the sufficient goods on the shelf and are closest to where they are. 
To meet the needs, it requires the application should have two features, real-time data handling and mobility support. And, the solution of using SAP HANA and SAPUI5 is the perfect match. In the combination of in-memory technology and column-oriented storage, applications powered by SAP HANA platform can provide an instant response to any request. And, if developed in SAPUI5, new Apps can run on all the UI5-compatible browsers across different devices with the similar user experiences.
Thus, in the following chapters, I will show you how to take advantage of the features of these two SAP products and give a step-by-step instruction about the implementation of the application. 


# Key Technologies:
This article is for the readers who are expected to have the basic knowledge of SAP BW and HANA products. To better understand the technologies to be covered, I will give a brief introduction, explaining their roles and how they are to be applied.

<ins>**SAP PI**</ins>

SAP PI is an SAP product which is widely used as the data exchange hub between SAP and non-SAP systems. In retail business, SAP PI usually plays a critical role of connecting SAP system to the external system, like POS device. In this case, as soon as the business transaction data get into the POS system, they will be posted to PI and then immediately transmitted to SAP BW system through Web Service. The whole process happens in Real-Time.

<ins>**BW-on-HANA**</ins>

SAP BW is still the primary business warehouse system used in most SAP landscapes. Today more and more customers start to migrate the BW system to HANA Database. Within BW powered by HANA, we are able to not only continue using the BW objects, like InfoObject and DSO and but start exploring the HANA’s new characteristics, like in-memory data processing and agile data modelling. Just as other databases, HANA supports functions, stored procedures and SQL(Script) operations. In additions, it boarded a lot of excited new technologies as detailed below.

<ins>**HANA Spatial Functions**</ins>

With the launch of HANA SP6, SAP delivers a set of functions to facilitate the operations on spatial information. With the new option, it becomes easy and quick to process the geometry data stored in HANA tables.  In this case, I will show how to use it to calculate the distance between two geo-points based on their longitude and latitude information. 


<ins>**HANA XS Server**</ins>

In addition to the mature applications in SAP BW, ERP or CRM, SAP brings in XS (Extended Application Services) into HANA platform to allow the native HANA applications to access the data in HANA database via HTTP protocol. In this case, a SAPUI5 application is built to consume the HANA data models.


<ins>**OData**</ins>

The native HANA applications usually access the HANA data exposed with OData(Open Data Protocol) service, which is a data access protocol initiated by Microsoft. It is designed to provide standard CRUD(Create, Read, Update, Delete) operation to a data source via a website. By its mechanism, the CRUD statements can be transferred over the (Restful) HTTP requests and get the results back to UI client in XML or JSON format.

<ins>**SAPUI5**</ins>

SAPUI5 is a UI development toolkit, which combines a Javascript library and a bound of pre-defined controls, supporting HTML5 and CSS. It is mostly used to develop Fiori Apps, the future SAP Apps that can run on all devices. In this case, I am going to build an App that will derive SAPUI5 mobile library.
Now let’s begin our real journey.

# Step-by-Step Approach:

## 1. Create BW DataSource for Real-Time data acquisition

![alt text](/images/F1.png?raw=true)

**Figure 1 – Real Time Data Acquisition**

First we need to build up a DataSource to receive the data posted from PI system through Web Service. In our scenario, the external system is the POS device installed in every store. Therefore, after each transaction, the sales data will get into the BW PSA table in Real-Time. 

![alt text](/images/F2.png?raw=true)

**Figure 2 – Datasource and the linked PSA table (“/BIC/B00040980000“)**

(There are some extra developments on PI and ABAP sides, requesting to set up the interfaces and build up the mapping out fields. Such details will not be covered in this article. You can find a lot of information from SAP SDN network.)

## 2. Create the data flow to get the stock snapshot data from ECC system

After every day’s business, the stock information of each shop are collected, replenished and updated in the statistics table - “S032” in ECC system. And then, during the night time, the daily snapshot data are extracted to BW system and will be used for the inventory reporting purpose.

![alt text](/images/F3.png?raw=true)

**Figure 3 – the BW data flow to load the stock snapshot data**

## 3. Prepare for the latitude and longitude information of each store  
In BW store master data table, I add two attribute fields to keep the latitude and longitude information.

![alt text](/images/F4.png?raw=true)

**Figure 4 – Store(Plant) GPS data**

## 4. Build a HANA Data Model   
SAP HANA supports three types of views - Attribute, Analytical and Calculation Views. I am going to build a Calculation View on the top of the existed BW tables.
What we want is the Real-Time stock data of each store. It is easy to get the number from the inventory data subtracted the Real-Time sales part. In HANA view, I create a calculation field named “QYT” **(= "Inventory - Quantity" - "RT Sales - Quantity")**

![alt text](/images/F5.png?raw=true)
**Figure 5 – Build HANA model to populate Real-Time Stock**

![alt text](/images/F6.png?raw=true)
**Figure 6 – Real-Time Stock of Material “000000000211110095” with the store GPS Information**

5. Create HANA procedure for spatial data calculation
We need to know the stores which have the stock and located within the distance from certain position. In the first step of the procedure we access the calculation view created in step 4 to get the all the stores that have the material in stock. Second, we call the HANA spatial functions “ST_POINT”, “POINT” and “ST_DISTANCE” to filter out those within the distance.
The below statement is to calculate the distance between the input position (v_lng and v_lat) and the GPS locations of all selected stores.

NEW ST_POINT('POINT('|| LNG || ' '|| LAT ||' )', 4326).ST_DISTANCE( NEW ST_POINT('POINT('|| :V_LNG || ' '|| :V_LAT ||')', 4326), ‘kilometer'))
