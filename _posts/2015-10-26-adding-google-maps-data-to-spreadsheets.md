---
layout: post
title:  "Geoanalysis in Google Sheets with Custom Functions and the Google Maps API"
date:   2015-10-26 12:10:00
categories: tutorials
tags: tutorials tech hacks
comments: true
excerpt_separator: <!--more-->
thumb: /images/maps-google-sheets/sheet-starter.png
slug: geoanalysis-google-sheets-custom-functions-google-maps-api
published: true
---

**Here's the scenario.** You're an independent clothing designer, thinking of opening a pop-up shop in Brooklyn. You know you've made a number of sales in Brooklyn in the last year, but want to do some quick geographic analysis to get a visual on the neighborhood distribution to help you figure out where you should open up shop.

**Our goal:**  To take a list of customer addresses and create a pie chart illustrating the distribution of neighborhoods.

<!--more-->

### Prerequisites

- Be able to write basic JavaScript
- Know what an API is
- Have access to a Google or Google Apps account
<br>

### What you'll learn in this tutorial

- Add a custom function to Google Sheets
- Interact with the Google Maps API via Google Apps Script
- Use Underscore.js from within Google Apps Script to parse the Maps API response
<br>

### Let's get started

We'll start with a list of addresses. To start, open up [this data sample](https://docs.google.com/spreadsheets/d/1SKRvclwjUWKuB6aZfPvHWewWHchRxpC2GmVXRC3LfBY/edit#gid=0) and make a copy.

![](/images/maps-google-sheets/sheet-starter.png)
{:center}

Our goal is to be able to fill out the Neighborhood column with neighborhoods like "Brooklyn Heights" and "Williamsburg".

In order to pull this data we will create a custom `=NEIGHBORHOOD(address)` function which we will be able to use from within a Google Sheets cell.

### Add a custom function to Google Sheets

From the list of addresses you just copied, access the **Tools** menu, then choose **Script Editor**. You'll see a pop-up menu with lots of helpful links, but for now, just click **Blank Project**.

Erase the existing code in the editor. Then add:

```javascript
function NEIGHBORHOOD(address) {
  return "My Neighborhood";
}
```

Any function we create in the editor here will automatically be available to our Google Sheets cells. Save your Script project and give it a name. Then go back to your spreadsheet, and try typing in `=NEIGHBORHOOD()`. Your cell should calculate even though we didn't actually provide an address, and return the value "My Neighborhood."

Unfortunately, you may have noticed that there was no auto-complete text that popped up like there normally is when you start typing a more common formula like `VLOOKUP` or `MAX`. We can fix that.

Directly above the function declaration in your Script, add the following JSDoc tag.

```javascript
/**
 * Returns the Google Maps political neighborhood for the given address.
 *
 * @param {string} address - The input address.
 * @return The neighborhood (political) for this address, based on the Google Maps API.
 * @customfunction
 */
```

If you're not familiar with JSDoc, it's a documentation generator for JavaScript. The code above is just a JavaScript comment, but it follows a specific formatting that JSDoc treats as documentation about the following function. In this case, we're providing information that will tell users how to use our custom function.

Save your script and give it a name. Then go back and type `=NEIGHBORHOOD()` into a cell in Google Sheets, and observe the magic.

![](/images/maps-google-sheets/formula-pop-up.png)
{:center}
<br>

### Get geographic information from the Google Maps API

Google Docs provides a `Maps` Service that gives us access to lots of methods for interacting with the Google Maps API. The beauty of using this directly from Apps Script is that we don't have to deal with getting an API key or doing any configuration whatsoever. We'll run through one of the basic uses of this service to get the neighborhood information.

Replace the contents of the `NEIGHBORHOOD` function with the below.

```javascript
function NEIGHBORHOOD(address) {
  var geocoder = Maps.newGeocoder();
  var location = geocoder.geocode(address).results[0];
  var components = location.address_components;
  var indexedComponents = _.indexBy(components,'types');

  var component = "neighborhood,political";

  return _.has(indexedComponents, component) ? indexedComponents[component].long_name: "Not Found"
}
```

If you try out this function, it won't work yet – we need to load the Underscore library, which we'll do in a moment.

*The TL;DR on what we just did with the code above:*

- Get access to the Geocoder
- Geocode the address
- Get the address components from the Google Maps results
- Turn the array into an indexed object (with Underscore)
- Return the neighborhood name, if it exists

Let's walk through each of these steps to explain what's going on (and install Underscore in the process).

###### **1. Get access to the Geocoder.**

First we get a new instance of a Maps geocoder object, which will let us actually parse an address and get information from it.

```javascript
  var geocoder = Maps.newGeocoder();
```

###### **2. Geocode the address.**

Then we get an object from the Google Maps API using that geocoder.

```javascript
  var location = geocoder.geocode(address).results[0];
```

`Geocoder.geocode(address)` will return an object that is structures as shown below. Inside this object, the `results` array will contain one more more resulting map objects with information about the address we looked up. If Google Maps returns more than one object in the `results` array, it's because there is some ambiguity as to the location the input `address` refers to.

In general, we will assume that the first element in the `results` array is the one we want to deal with, so we call `.results[0]` to narrow down our response.

```javascript
{
  "results": [
    {
      "formatted_address":  "...",
      "types":              [...],
      "geometry":           {...},
      "address_components": [...],
      "place_id":           "..."
    },
    ...
  ],
  "status": "OK"
}
```

###### **2. Parse the `address_components`.**

Each element inside the `results` array has a number of interesting properties. For our analysis, we only need to deal with `address_components`, which breaks down our address into properties like its neighborhood, zip code, etc.

```javascript
  var components = location.address_components;
```

Here's what `address_components` looks like when we input the address for **Brooklyn Bowl** (61 Wythe Ave, Brooklyn, NY 11249).

```javascript
"address_components": [
  {
    "types": [
      "street_number"
    ],
    "short_name": "61",
    "long_name": "61"
  },
  {
    "types": [
      "route"
    ],
    "short_name": "Wythe Ave",
    "long_name": "Wythe Avenue"
  },
  {
    "types": [
      "neighborhood",
      "political"
    ],
    "short_name": "Williamsburg",
    "long_name": "Williamsburg"
  },
  {
    "types": [
      "sublocality_level_1",
      "sublocality",
      "political"
    ],
    "short_name": "Brooklyn",
    "long_name": "Brooklyn"
  },
  {
    "types": [
      "administrative_area_level_2",
      "political"
    ],
    "short_name": "Kings County",
    "long_name": "Kings County"
  },
  {
    "types": [
      "administrative_area_level_1",
      "political"
    ],
    "short_name": "NY",
    "long_name": "New York"
  },
  {
    "types": [
      "country",
      "political"
    ],
    "short_name": "US",
    "long_name": "United States"
  },
  {
    "types": [
      "postal_code"
    ],
    "short_name": "11249",
    "long_name": "11249"
  }
]
```

If you look closely, you'll notice that this is an array of objects, each of which map to specific types of information. We want to get an object like the one below, but because it lives in an array, we can't simply look this up by a key. Instead, we would need to loop through every array element until we find the type we're looking for.

```javascript
{
  "types": [
    "neighborhood",
    "political"
  ],
  "short_name": "Williamsburg",
  "long_name": "Williamsburg"
}
```

###### **PITSTOP: Load Underscore into your project**

Underscore.js makes tackling the above challenge a little cleaner and easier.

The Underscore.js library provides a bunch of utility functions that simplify many things JavaScript developers find themselves wishing JavaScript did natively. Many of these tasks involve manipulating arrays and objects to work the way we want them to. These functions are accessible via the `_` object (yes, it is allowable to use a single underscore as a variable name in JavaScript).

Let's load in the Underscore library so that we can make use of these functions.

  - In your Script Editor, select **File** > **New** > **Script File**.
  - Name it `Underscore.gs`, although the name doesn't really matter.
  - Delete the default contents of your new file
  - Copy the contents of this [minified version](http://underscorejs.org/underscore-min.js) of Underscore.js into your new file
  - Save

###### **4. Turn the array into an indexed object**

Now, all of the Underscore functions are available to you in all of your Script files. The first function we're going to use is `_.indexBy`. Straight from the [Underscore documentation](http://underscorejs.org/#indexBy):

**indexBy** &nbsp;&nbsp;&nbsp;`_.indexBy(list, iteratee, [context])`

"Given a **list**, and an **iteratee** function that returns a key for each element in the list (or a property name), returns an object with an index of each item."

This sounds a little more confusing than it needs to be. In our case, all we're doing is passing an array called `components` to `_.indexBy(components,'types')` and asking it to look at each object in that array for a `types` property. It will then map the entire array into a new object of key-value pairs, where each key is the value of the `types` property in the original object, and the value is the object itself.

Note that the `types` property in each of the original address components array were themselves arrays, but keys in JavaScript can't be arrays, so the they have been converted to strings in the result below.

Now that we have an object of key-value pairs, we can look up our neighborhood by its key – in our case we want `"neighborhood,political"`.

```javascript
{
  "street_number": {
    "types": [
      "street_number"
    ],
    "short_name": "61",
    "long_name": "61"
  },
  "route": {
    "types": [
      "route"
    ],
    "short_name": "Wythe Ave",
    "long_name": "Wythe Avenue"
  },
  "neighborhood,political": {
    "types": [
      "neighborhood",
      "political"
    ],
    "short_name": "Williamsburg",
    "long_name": "Williamsburg"
  },
  "sublocality_level_1,sublocality,political": {
    "types": [
      "sublocality_level_1",
      "sublocality",
      "political"
    ],
    "short_name": "Brooklyn",
    "long_name": "Brooklyn"
  },
  "administrative_area_level_2,political": {
    "types": [
      "administrative_area_level_2",
      "political"
    ],
    "short_name": "Kings County",
    "long_name": "Kings County"
  },
  "administrative_area_level_1,political": {
    "types": [
      "administrative_area_level_1",
      "political"
    ],
    "short_name": "NY",
    "long_name": "New York"
  },
  "country,political": {
    "types": [
      "country",
      "political"
    ],
    "short_name": "US",
    "long_name": "United States"
  },
  "postal_code": {
    "types": [
      "postal_code"
    ],
    "short_name": "11249",
    "long_name": "11249"
  }
}
```

###### **5. Return the neighborhood, if it exists**

Finally, we use another Underscore method `_.has(object, key)` to determine if our `indexedComponents` objects contains the key we're looking for.

```javascript
var component = "neighborhood,political";

return _.has(indexedComponents, component) ? indexedComponents[component].long_name : "Not Found"
```

If the results contain a "neighborhood,political" property, we'll get the `long_name` property inside. Otherwise we return "Not Found." Since we're dealing with addresses in the same part of the world, we're going to hope that all addresses have this component. In reality, the various address components that Google returns vary based on locale. You can always come back and choose a different address component later if you need to add complexity.

Note: If you haven't seen this kind of construct before, it's called a **[ternary operator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Conditional_Operator)**.
<br>

### Finish your neighborhood analysis

The last step is to go back and test out our function on the original dataset, and generate a pie chart.

![](/images/maps-google-sheets/sheet-result.png)
{:center}

![](/images/maps-google-sheets/chart-result.png)
{:center}

Looks like Williamsburg is the clear winner for our pop-up shop!
<br>

### For further information

- The full documentation on the Google Maps API for Apps Script is [here](https://developers.google.com/apps-script/reference/maps/).

<br><br>
