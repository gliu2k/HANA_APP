# Description

This is an article that I wrote in **year 2016** about how to utilize the new features of SAP HANA. And this is the 2nd article that I submitted to SAPexperts.

![alt text](/images/F0.png?raw=true)

# Background
E-commerce is becoming a trend and getting more and more popular in retail business. However, certain products, like shoes or clothes, still need the involvement of customer’s personal experience. Such generates a brand-new business concept called O2O, in terms of online-to-offline, which is a strategy to draw customers from the online channels to the physical stores. And, to access online information, there are various approaches. Nowadays, the most popular and efficient way is through smart phones. In this article, I will demonstrate an application that achieves these two business models. 

# Business Case
The O2O concept and mobile Apps can be adopted in many requirements of today’s business. For instance, in all walks of business, running campaign is the common way to promote products. Today, one of the most efficient, or cheapest, methods to launch a campaign is through social media, email or SMS. Let’s suppose a fashion footwear company wants to put a new product onto the market. It publishes an advertisement on social media, sending the message to the targeted customers. People who see the information on their phone may want to go to the store and pick a fitted pair immediately. Usually, they prefer to go to those stores that have sufficient goods on the shelf and are closest to where they are. 
To meet the needs, the application should have two features, real-time data handling and mobility support. And the solution of using SAP HANA and SAPUI5 is the perfect match. In the combination of in-memory technology and column-oriented storage, applications powered by SAP HANA platform can provide an instant response to any request. And, if developed in SAPUI5, new Apps can run on all the UI5-compatible browsers across different devices with similar user experiences.
Thus, in the following chapters, I will show you how to take advantage of the features of these two SAP products and give a step-by-step instruction about the implementation of the application. 

# Key Technologies
This article is for the readers who are expected to have basic knowledge of SAP BW and HANA products. To better understand the technologies to be covered, I will give a brief introduction, explaining their roles and how they are to be applied.

<ins>**SAP PI**</ins>

SAP PI is an SAP product which is widely used as the data exchange hub between SAP and non-SAP systems. In retail business, SAP PI usually plays a critical role in connecting SAP system to the external system, like POS device. In this case, as soon as the business transaction data gets into the POS system, they will be posted to PI and then immediately transmitted to SAP BW system through Web Service. The whole process happens in Real-Time.

<ins>**BW-on-HANA**</ins>

SAP BW is still the primary business warehouse system used in most SAP landscapes. Today more and more customers start to migrate the BW system to HANA Database. Within BW powered by HANA, we can not only continue using the BW objects, like InfoObject and DSO and but start exploring the HANA’s new characteristics, like in-memory data processing and agile data modelling. Just as other databases, HANA supports functions, stored procedures and SQL(Script) operations. In addition, it includes a lot of excited new technologies as detailed below.

<ins>**HANA Spatial Functions**</ins>

With the launch of HANA SP6, SAP delivers a set of functions to facilitate the operations on spatial information. With the new option, it becomes easy and quick to process the geometry data stored in HANA tables.  In this case, I will show how to use it to calculate the distance between two geo-points based on their longitude and latitude information. 


<ins>**HANA XS Server**</ins>

In addition to the mature applications in SAP BW, ERP or CRM, SAP brings in XS (Extended Application Services) into HANA platform to allow the native HANA applications to access the data in HANA database via HTTP protocol. In this case, a SAPUI5 application is built to consume the HANA data models.


<ins>**OData**</ins>

The native HANA applications usually access the HANA data exposed with OData(Open Data Protocol) service, which is a data access protocol initiated by Microsoft. It is designed to provide standard CRUD(Create, Read, Update, Delete) operation to a data source via a website. By its mechanism, the CRUD statements can be transferred over the (Restful) HTTP requests and get the results back to UI client in XML or JSON format.

<ins>**SAPUI5**</ins>

SAPUI5 is a UI development toolkit, which combines a Javascript library and a bound of pre-defined controls, supporting HTML5 and CSS. It is mostly used to develop Fiori Apps, the future SAP Apps that can run on all devices. In this case, I am going to build an App that will derive SAPUI5 mobile library.

**Now let’s begin our real journey.**

#  Step-by-Step Approach:

# 1. Part I: On-premise

## 1.1 Create BW DataSource for Real-Time data acquisition

![alt text](/images/F1.png?raw=true)

**Figure 1 – Real Time Data Acquisition**

First, we need to build up a DataSource to receive the data posted from PI system through Web Service. In our scenario, the external system is the POS device installed in every store. Therefore, after each transaction, the sales data will get into the BW PSA table in Real-Time. 

![alt text](/images/F2.1.png?raw=true)

![alt text](/images/F2.2.png?raw=true)

**Figure 2 – Datasource and the linked PSA table (“/BIC/B00040980000“)**

There are some extra developments on PI and ABAP sides, requesting to set up the interfaces and build up the mapping out fields. Such details will not be covered in this article. You can find a lot of information from SAP SDN network.

## 1.2. Create the data flow to get the stock snapshot data from ECC system

After every day’s business, the stock information of each shop is collected, replenished and updated in the statistics table - “S032” in ECC system. And then, during the nighttime, the daily snapshot data are extracted to BW system and will be used for the inventory reporting purpose.

![alt text](/images/F3.png?raw=true)

**Figure 3 – the BW data flow to load the stock snapshot data**

## 1.3. Prepare for the latitude and longitude information of each store  
In BW store master data table, I add two attribute fields to keep the latitude and longitude information.

![alt text](/images/F4.png?raw=true)

**Figure 4 – Store(Plant) GPS data**

## 1.4. Build a HANA Data Model   
SAP HANA supports three types of views - Attribute, Analytical and Calculation Views. I am going to build a Calculation View on the top of the existing BW tables.
What we want is the Real-Time stock data of each store. It is easy to get the number from the inventory data subtracted from the Real-Time sales part. In HANA view, I create a calculation field named “QYT” **(= "Inventory - Quantity" - "RT Sales - Quantity")**

![alt text](/images/F5.png?raw=true)

**Figure 5 – Build HANA model to populate Real-Time Stock**

![alt text](/images/F6.png?raw=true)

**Figure 6 – Real-Time Stock of Material “000000000211110095” with the store GPS Information**

## 1.5. Create HANA procedure for spatial data calculation
We need to know the stores which have stock and located within the distance from certain position. In the first step of the procedures we access the calculation view created in **step 4** to get the all the stores that have the material in stock. Second, we call the HANA spatial functions “ST_POINT”, “POINT” and “ST_DISTANCE” to filter out those within the distance.

The below statement is to calculate the distance between the input position (v_lng and v_lat) and the GPS locations of all selected stores.

```
NEW ST_POINT('POINT('|| LNG || ' '|| LAT ||' )', 4326).ST_DISTANCE( NEW ST_POINT('POINT('|| :V_LNG || ' '|| :V_LAT ||')', 4326), ‘kilometer'))
```

![alt text](/images/F7.png?raw=true)

**Figure 7 – HANA procedure to call spatial functions** 
```
BEGIN 

/* Get all the stores that have the material in stock (Real-Time) */
LOC = SELECT PLANT, NAME, QTY, TO_DECIMAL(LONGITUDE) AS LNG, TO_DECIMAL(LATITUDE) AS LAT
	 			FROM "_SYS_BIC"."YSANDBOX/ZCV_STORE_STOCK" 
	 			WHERE QTY > 0 AND LONGITUDE != '' AND LATITUDE != '' 
AND MATERIAL = :V_MAT;

/* Get the stores within certain distance from the input GPS location	*/		
LT_STOCK = SELECT  PLANT, NAME, QTY, LNG, LAT
	 			FROM :LOC
	 			WHERE (NEW ST_POINT('POINT('|| LNG || ' '|| LAT ||' )', 4326).ST_DISTANCE( NEW ST_POINT('POINT('|| :V_LNG || ' '|| :V_LAT ||')', 4326), ‘kilometer')) < :V_DIST;

```
> [!NOTE] 
> SRID 4326 is referring to WGS84, the standard provides a spheroidal reference surface for the Earth. It is the spatial reference system used by the Global Positioning System (GPS).

## 1.6. Create a SQLScript Calculation View 
HANA partial functions support specific data type, for instance decimal used as the GPS position. However, it is impossible to specify the data type for the parameters passed from web browser. Therefore, we need to create a SQLScript Calculation View to perform data-parsing and conversion. I created a dedicated package for the XS application called “XSPJ1”.  Under it, I put this view and OData service definition file.

![alt text](/images/F8.png?raw=true)

**Figure 8 – HANA SQLScript Calculation View**

```
BEGIN 
  declare items varchar(100) ARRAY;
  declare _text varchar(100);
  declare _index integer;
  
  declare v_mat nvarchar(100);
  declare v_lng decimal(10,6);
  declare v_lat decimal(10,6);
  declare v_dist int;
  
  _text := :v_param;
  _index := 1;

/* parse the parameter and covert data type */ 
  WHILE LOCATE(:_text,',') > 0 DO
	items[:_index] := SUBSTR_BEFORE(:_text,',');
	_text 	:= SUBSTR_AFTER(:_text,',');
	_index	:= :_index + 1;
  END WHILE;
  items[:_index] := :_text;
  
  v_mat := :items[1];
  v_lng := to_decimal(:items[2]);
  v_lat := to_decimal(:items[3]);
  v_dist := to_int(:items[4]);

/* call the stored procedure by passing the parameters */
  CALL "_SYS_BIC"."YSANDBOX/ZDP_STORE_STOCK"(:v_mat,:v_lng,:v_lat,:v_dist,var_out);

END 

```
## 1.7. Create XSODATA service
Now, it’s time to create the XSODATA service. Let’s do it in HANA WEB-IDE Editor.  Open web browser, go to the URL(http://host:8000/sap/hana/ide/) defined in HANA system and input the logon data.

![alt text](/images/F9.1.png?raw=true)

Click “Editor” ICON

![alt text](/images/F9.2.png?raw=true)

**Figure 9 – HANA procedure to call spatial functions** 

Create sub-package “services” under package “XSPJ1” and add the service that consumes the HANA SQLScript view created before.

![alt text](/images/F10.png?raw=true)

**Figure 10 – Create ODATA Service** 

Now we can test the service to see whether we are able to get the data. 

<ins>http://HANADEV:8000/XSPJ1/services/stock.xsodata/InputParams(v_param ='000000000211110095,40.757749,-73.985973,10')/Results?$select=*&$format=json</ins>

![alt text](/images/F11.png?raw=true)

**Figure 11 – OData retrieved from HANA model (same as Figure 6)**

So far, the development of the on-promise side has been completed. Let’s start to create the UI.

# 2. PART II: On-Demand
I am going to use SAPUI5 to create an App to consume the data in the HANA database via OData service. 

![alt text](/images/F12.png?raw=true)

**Figure 12 – Design Architecture**

## 2.1. IDE preparation  
Let’s download the latest Eclipse release - MARS and install the plugin of SAPUI5 toolkit from https://tools.hana.ondemand.com/mars

![alt text](/images/F13.1.png?raw=true)

![alt text](/images/F13.2.png?raw=true)

**Figure 13 – Download the UI5 toolkit**

## 2.2. Create a SAPUI5 project 
Follow the wizard, use mobile library “sap.m” and choose XML(view type).

![alt text](/images/F14.1.png?raw=true)

![alt text](/images/F14.2.png?raw=true)

![alt text](/images/F14.3.png?raw=true)

![alt text](/images/F14.4.png?raw=true)

**Figure 14 – Create a SAPUI5 project**

In SAPUI5 development, it is recommended to apply the MVC (Model View Controller) architecture into the design in order to make the functions independent. 

![alt text](/images/F15.png?raw=true)

**Figure 15 – SAPUI5 MVC Architecture**

So, I will follow the pattern and apply it in the development of this App. Please download the source files from Github and import them into the created Eclipse project. And we will view the structure under “WebContent” folder.

**Model:**

It provides methods to retrieve the data from the database and to set and update data. In this case, I directly put in the mock data for the product and GPS of customer’s position to simply the simulation. And, by default, the distance is 10(km).

![alt text](/images/F16.png?raw=true)

**Figure 16 – Component.js**

**View:**

It contains the logic responsible for defining and rendering the UI. In this case, it loads the predefined views “Search” and “Result”.
![alt text](/images/F17.png?raw=true)

**Figure 17 – App.view.js**

**Controller:**

It offers different methods to handle user input and interactions, process incoming requests and execute appropriate logic. In this case, it is separated into three controllers based on their functions.

In the “Search” controller, it receives the “distance” value input by customer and navigates to the result controller.

![alt text](/images/F18.png?raw=true)

**Figure 18 – Search.controller.js**

 “App” controller is responsible for passing the control from “Search” controller to “Result” controller. 
 
![alt text](/images/F19.png?raw=true)

**Figure 19 – App.controller.js**

In the “Result” controller, it collects the material, distance, GPS of customer’s location data and makes them up into a parameter for the CRUD query. Then, it calls Ajax and passes the CRUD query to HANA OData service via HTTP request. At last, it displays a Google Map with the searched results on it.

![alt text](/images/F20.png?raw=true)

**Figure 20 – Result.controller.js**

# 3. Part III: Scenario Simulation
We know how it is designed. Now let’s see how it works. 

As mentioned, the App is using “sap.m” library that supports mobile device. In theory, the SAPUI5 App can run across different devices. To simplify the simulation, I choose Google Chrome browser in PC environment to run the App. 

I deploy the App on Apache Tomcat installed in local PC and call it in Chrome browser. In the “Search” view, it shows the promoted product and the distance scope for input.

![alt text](/images/F21.png?raw=true)

**Figure 21 – Customer see the product**

Customer changes the default value of 10km to 1km, click “GO” button. The result will be shown on the fly.

In Google Map, the arrow icon points to the position where the customer is. As the search result, it shows only two stores, 8001 and 8002, which are within 1km from the customer. And, if the icon of the store is clicked, the Real-Time store stock data will pop up. Customer can decide which store to go to.

![alt text](/images/F22.1.png?raw=true)
![alt text](/images/F22.2.png?raw=true)

**Figure 22 - Customer see the search results**

If using the default distance of 10km to check the stores, we can find that store 8003 on 5th Ave with 6PC in stock shown up as well. But it may be a bit too far for this customer.

![alt text](/images/F23.png?raw=true)

**Figure 23 - Customer see a different search result with different distance input**


> [!NOTE]
> I deploy the App in local host and try to access the resource on the HANA XS server. Due to security reasons, most browsers disallow such cross-domain requests. To resolve it, you can install the "Allow-Control-Allow-Origin" plugin for Chrome. 

![alt text](/images/F24.png?raw=true)

**Figure 24 - Chrome Plugin**

# 4. Summary
In this article, I have demonstrated how to build an SAPUI5 App to retrieve the Real-Time data from SAP HANA. It presents a way about how to leverage the features like modelling, spatial functions and XS in SAP HANA platform, as well as how to combine them with the traditional functionalities in SAP BW product.



