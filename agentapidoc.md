# VeriSmart Agent API Documentation V1 

This documentation is intended for software developers to allow them to query the VeriSmart booking database within their own applications.

## Introduction

The VeriSmart Agent API (Referred to as just "API" hereafter) currently has two endpoints available.

The "Provisioning" API allows a developer to fetch and store the credentials needed in JSON format using the Agents username and password. 

The "Booking" API allows you to use these credentials to fetch, and insert data in JSON format from our database.

## Conventions

All responses include a field of "response_status".

Response of 1 means the query is successful, and 0 means failure. If failed, the only other field will be "error", which contains the reason for failure.

All submitted data should be URL Encoded.

All responses will be JSON encoded.

All endpoints use SSL, and will not work over a normal http:// transport.

## Endpoints

### Provisioning

    https://www.verismart.co.uk/dcal/api/v1/provisioning/

### Booking

    https://www.verismart.co.uk/dcal/api/v1/booking/

## Provisioning API

The Provisioning API allows you to fetch the apikey and secret.

Three fields are submitted, either via POST or GET

    "devkey" - your developers key, supplied separately
    "username" - Agents calendar username, usually an email address
    "password" - Agents calendar password

All GET requests should be URL Encoded.

As all of our Agents currently login to the calendar and should know their login details, the recommended flow is to request their login details and use them to query the Provisioning API.

V2 of the Provisioning API will use the OAuth, or a similar system as yet to be decided.

### Return Data

JSON-encoded string with request_status, apikey and secret.

Line breaks added for clarity

    {
	"request_status":"1",
	"apikey":"D15AD8F8015989F637EF53E8A1458YHR4C444B58",
    "secret":"OTc2ZDg3ODFlMmU4MzRjNzBjODgwZWFmZGI4YjZFgrrtgGMyODc4MWE1NTk5Njg0NGY4ZjQxNzVjNWMxODJiZA%3D%3D"
	}

You will need both the apikey and the secret to interact with the Booking API.

## Booking API

The booking API uses signed query strings to authenticate the request. The querystring will be signed using the secret, which is then appended to the URL before the GET request is made.

### Building a signed URL

We will be signing just the query string, which contains the apikey and our requests and/or data.

An simple example query string, showing all bookings for today:

    apikey=D15AD8F8015989F637EF53E8A1458YHR4C444B58&mode=show

Now we take that string, and and hash it with sha512 to produce our signature string.

In PHP:

    $signature = hash_hmac("sha512", $query_string, $secret);

This signature can now be appended as "signature" to the url

In PHP:

    $query_string = $query_string . '&signature=' . $signature;

And this URL can be used to query the API. The signature tells VeriSmart that this request is correctly authed, and SSL ensures the request data remains private.

A full PHP example:

    		$request_data = array(
    		'apikey' => $apikey,
    		'mode' => 'show'
    		);
    		
    		$query_string = http_build_query($request_data);
    				
    		$signature = hash_hmac("sha512", $query_string, $secret);
    	
    		$query_string = $query_string . '&signature=' . $signature;
    						
    		$result = file_get_contents($apiurl . '?' . $query_string);

## Booking API Query Types

The Booking API currently supports querying on specific bookings, bookings on a specific day, bookings in a specific month, and bookings on a specific property.

We will start with the most useful query type, specific property.

### Fetch bookings by specific property

The mode is "property", and we are basically querying the VeriSmart database with a search.

The search fields available are

    property_name - This is the building name, eg Heron Court
    property_flatno - Can be numbers, letters or mixed
    property_houseno - Can be numbers, letters or mixed
    property_address1 - Address Line 1, traditionally the street address
    property_address2 - Address Line 2, rarely used
    property_town - The postal town
    property_county - The postal county
    property_postcode - REQUIRED

The property search will find a single property that matches the criteria, and return all of it's bookings for the authed customer. Most fields are optional, but using more data will ensure more accurate results.

Example url, linebreaks added for clarity:

    ?apikey=APIKEY&mode=property&property_flatno=98
    &property_houseno=99&property_name=Bakewell%20Mansions
    &property_address1=Bobby%20Tables%20Street&property_postcode=BH11XX&signature=SIGNATURE

##### A Note on Postcodes

All spaces in submitted postcodes are ignored, so feel free to submit with or without spaces.

## Fetch bookings on specific day

To select all bookings from a specific day, the mode is "show" and you supply three fields.

The fields are:
    
    day - numeric, 1..31
    month - Month name, January...December
    year - four digits, 2013
    
Example url:

    ?apikey=APIKEY&mode=show&month=April&year=2013&day=9&signature=SIGNATURE

## Fetch Bookings on specific month

To select all bookings from a specific month, the mode is "show" and you supply two fields.

The fields are:

    month - Month name, January...December
    year - four digits, 2013

Example url:

    ?apikey=APIKEY&mode=show&month=April&year=2013&signature=SIGNATURE

## Fetch available time slots on a specific day

To select available slots, the mode is "slots" and you supply three fields.

The fields are

    day - 1...31, day number without leading zeroes
    month - Month name, January...December
    year - four digits, eg 2013

Example url:

    ?apikey=APIKEY&mode=slots&day=8&month=April&year=2013&signature=SIGNATURE

Successful Return:

    {
      "request_status": "1",
      "0": {
    "teamid": "1506",
    "team_name": "Deans Test Inventory Clerk",
    "available": 1,
    "status": "Available",
    "available_slots": [
      "Anytime",
      "Morning",
      "Afternoon",
      "TBA",
      "09:00",
      "09:30",
      "12:30",
      "16:30",
      "17:00"
    ]
      }
    }

Note that certain logins scope with more than one "team" within the VeriSmart booking system. As you can see from the json returned,this API call can return multiple teams availability.

The following fields are returned for each team record:

    teamid: Unique ID for that team
    team_name: Business name for that team
    available: Boolean, if "0" then that team is not available that day
    status: "Available" or an explanation of why the team is not available, eg "Day is full"
    available_slots: an array of available slots. Slot names can be used as-is during API calls when creating bookings


## Create New Booking

The mode for creation is "create".

The following fields are accepted. Required fields are marked with an asterisk.

        flat_no - Flat Number/Letter
    house_no - House Number
    property_name - Property Name
    * address1 - First line of address, usually street name
    address2 - Second line of address, generic
    * town - Name of town
    county - Name of County
    * postcode - Postcode of property
    * bedrooms - Number of bedrooms in property
    * furnishings - UNFURNISHED, FURNISHED or PART-FURNISHED
    garden - Yes, No, Unknown
    outhouse - Yes, No, Unknown
    garage - Yes, No, Unknown
    * assignment - One of: INVENTORY, CHECK-IN, INTERIM, CHECK-OUT, UPDATE
    * day - Day of the month, eg 1 or 16
    * month - Textual month name, eg January, May
    * year - Four-digit format, eg 2013, 2015
    * service_time - Same as slots data, 24-format eg 09:00, 14:00
    * key_location - Where are the keys, One of: Agent, Tenant, Landlord, Property, Other
    contact - Contact name/number if meeting anyone
    branch_id - Not used yet
    comments - Free text field for any extra information
    tenant_name - Required for Paperless
    tenant_mobile - Required for Paperless
    tenant_email - Required for Paperless
    landlord_email - Optional for Paperless
    ondelivery_hook_url - A full URL which the system will POST json data to, on report delivery, see below for details

Please note, VeriSmart Tenant Delivery is optional, but all three tenant_ fields are required if you do wish to have us deliver to the Tenant.

Example url:


    https://www.verismart.co.uk/dcal/api/v1/booking/?apikey=APIKEY&mode=create&flat_no=9&house_no=23
    &address1=bellington+gardens&town=bournemouth
    &postcode=BH1+1XX&bedrooms=3&furnishings=UNFURNISHED&assignment=INVENTORY&
    day=26&month=May&year=2013&service_time=Anytime&key_location=Tenant&contact=07854123456&
    comments=Mind+the+dog&tenant_name=Dean+Owen&tenant_mobile=07854123456
    &tenant_email=dean%40de4n.co.uk&landlord_email=noone%40surrounded.org.uk&signature=SIGNATURE

### Returns

Status code, unique VeriSmart reference number

    {"request_status":"1","taskid":79172}

### ondelivery\_hook\_url

When the supplier has uploaded the report (which sends the report as a link via email), the system will POST the following data, in json format -

    taskid - Unique reference number
    report_url - URL to download the report PDF

Please note your URL can include query string information, but this will be converted to JSON and included within the payload.

The POST will contain a single parameter, called "payload", which will contain the JSON data, plus a field called "custom_data" which will contain any query string data included in the hook url.

Please note, the report may be uploaded multiple times, therefore this event may fire multiple times.

### Example Data

    {
	  "taskid": "123456",
      "report_url": "http:\/\/ivcn.saveatree.co.uk\/333872_79564.pdf",
      "custom_data": {
    	"mode": "new_upload",
    	"data": "myinternalref"
      }
    }
