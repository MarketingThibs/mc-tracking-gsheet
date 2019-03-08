# Bring Marketing Cloud Tracking Into GoogleSheet

## Intro

The idea here is to use Marketing Cloud *Cloud Pages* to display email tracking according the URL parameters that we fill. From a Google Sheet we can import the JSON from this URL. In other words we will be using *Cloud Page* as a JSON feed

## Building the Cloud Page

### Landing Page Creation

1. Create a landing page in Marketing Cloud
2. In the HTML code view, paste the source code (or your own version)
3. Save and publish the landing page

### Code Logic

The logic is to display different metrics according the parameters values. 

- if my `parammetric` = `all` then display generic metrics information
- if my `parammetric` = `click` then display click and links specific tracking information
- if my `parammetric` = `date` then display date intervall based tracking information
  - Retrieve another parameter `date1` as *start date* `date2` as *end date*
  - Retrieve another parameter (`dtrack`) that indicated with metric you would like to display for this interval 
    - send
    - click
    - open
    - bounce
    - unsubscribe

###  Code detailed

```html
<script runat="server">
  Platform.Load("Core", "1.1.1");
  // example URLs 
  // http://example.com/?tskey=41053&metric=date&date1=02-22-2019&date2=03-01-2019&dtrack=send
  // http://example.com/?tskey=41053&metric=all
  // http://example.com/?tskey=41053&metric=click
  try {
      // Get URL parameters values
      var paramtskey = Request.GetQueryStringParameter("tskey");
      var parammetric = Request.GetQueryStringParameter("metric");

      if (parammetric === "all") {
          // initialiwing the trigger send info thanks to the key we retrieve from the URL
          var tsd = TriggeredSend.Init(paramtskey);
          var triggerSendTracking = tsd.Tracking.Retrieve();
          var tsTracking = Stringify(triggerSendTracking);
          // use the following if you want to remove the brackets from the json
          //var outputmetric = tsTracking.replace(/\[/g, "").replace(/\]/g, "");

          // display something if the retrurn json is empty in case there is no activity for a like for journeys for instance
          // why tsTracking.length > 2 ? If you keep the brackets and the JSON is empty it will output [] only
          if (tsTracking.length > 2) {
            var outputmetric = Stringify(triggerSendTracking);
          } else {
            var outputmetric = '{"CustomerKey":"' + paramtskey + '"}';
          } 
      } else if (parammetric === "click") {
          var tsd = TriggeredSend.Init(paramtskey);
          var clickstsd = tsd.Tracking.Clicks.Retrieve();
          var tsTracking = Stringify(clickstsd);
          if (tsTracking.length > 2) {
            var outputmetric = Stringify(clickstsd);
          } else {
            var outputmetric = '{"CustomerKey":"' + paramtskey + '"}';
          } 
      } else if (parammetric === "date") {
          var tsd = TriggeredSend.Init(paramtskey);
          // retrieve start, end date and the type of tracking that we want for this intervall
          var date1 = Request.GetQueryStringParameter("date1");
          var date2 = Request.GetQueryStringParameter("date2");
          var dtrack = Request.GetQueryStringParameter("dtrack");
          //format: 'Click', '07-01-2019', '07-31-2019', 'day'
          // Valid values include Send, Open, CLick, Bounce, and Unsubscribe.
          if (dtrack === "send") {
              var sendsstsd = tsd.Tracking.TotalByInterval.Retrieve('Send', date1, date2, 'day');
              var outputmetric = Stringify(sendsstsd);
          } else if (dtrack === "click") {
              var clickstsd = tsd.Tracking.TotalByInterval.Retrieve('Click', date1, date2, 'day');
              var outputmetric = Stringify(clickstsd);
          } else if (dtrack === "open") {
              var openstsd = tsd.Tracking.TotalByInterval.Retrieve('Open', date1, date2, 'day');
              var outputmetric = Stringify(openstsd);
          } else if (dtrack === "bounce") {
              var bouncessstsd = tsd.Tracking.TotalByInterval.Retrieve('Bounce', date1, date2, 'day');
              var outputmetric = Stringify(bouncessstsd);
          } else if (dtrack === "unsubscribe") {
              var unsubscribestsd = tsd.Tracking.TotalByInterval.Retrieve('Unsubscribe', date1, date2, 'day');
              var outputmetric = Stringify(unsubscribestsd);
          } else {
            var outputmetric = '{"Name":"0","Unique":"0"}';
          }
      } else {
          // return something if there is nothing for the Google Sheet, so it is not displaying "error" 
          var outputmetric = '{"Name":"0","Unique":"0"}';
      }
      Write(outputmetric);
  } catch (ex) {
      Write("error message: " + ex);
  }
  </script>
```



### Example of URLs

-  Getting tracking based upon a date intervall 
  `http://example.com/?tskey=41053&metric=date&date1=02-22-2019&date2=03-01-2019&dtrack=send`
- Generic Tracking 
  `http://example.com/?tskey=41053&metric=all`
- Click and links specific tracking `http://example.com/?tskey=41053&metric=click`



## Importing the feed results

### Create the Import JSON function

For this part you can refer to [paulgambill's Google Sheet Script](https://gist.github.com/paulgambill/cacd19da95a1421d3164).

1. Create a new Google Spreadsheet.
2. Click on Tools -> Script Editor.
3. Click Create script for Spreadsheet.
4. Delete the placeholder content and paste the code from [this script](https://gist.github.com/paulgambill/cacd19da95a1421d3164).
5. Rename the script to ImportJSON.gs and click the save button.
6. Back in the spreadsheet, in a cell, you can type `=ImportJSON()` and begin filling out it’s parameters.

### Import your JSON feed into the Google Sheet

1. Select the metric that you want to display, we will take the example output for the following URL `http://example.com/?tskey=41053&metric=all`

```json
{"Client":{"ID":7276191},"CustomerKey":"1234key","Name":"journey-email-name","ObjectID":"s0me1d","LastSent":"2019-02-21T03:10:42.523","Sends":{"Total":115},"Bounces":{"Total":2,"HardBounces":1,"SoftBounces":0,"BlockBounces":1,"TechnicalBounces":0,"UnknownBounces":0},"Clicks":{"Total":3,"Unique":2},"Opens":{"Total":41,"Unique":28},"Unsubscribes":{"Unique":0}}
```

2. We have several options to display it into the sheet.
   - **Import Every thing**
      `==ImportJSON("http://example.com/?tskey=48088&metric=all"`
   - **Parse the JSON from the Gsheet**
     `=ImportJSON("http://cloud.mail.wefox.com/param-script?tskey=48088&metric=all", "/CustomerKey,/Name,/LastSent,/Sends/Total,/Clicks/Unique,/Opens/Unique,/Unsubscribes/Unique","noInherit,noTruncate,noHeaders"`
   - You can find [more info on the *Import JSON* features here](https://medium.com/@paulgambill/how-to-import-json-data-into-google-spreadsheets-in-less-than-5-minutes-a3fede1a014a) 

![](/GsheetImage.png)



## Resources

- [SSJS GetQueryStringParameter](https://developer.salesforce.com/docs/atlas.en-us.noversion.mc-programmatic-content.meta/mc-programmatic-content/ssjs_utilitiesGetQueryStringParameters.htm)
- [SSJS If Statment](https://www.w3schools.com/js/js_if_else.asp)
- SSJS Tirggerd Send Tracking
  - [Clicks.Retrieve](https://developer.salesforce.com/docs/atlas.en-us.noversion.mc-programmatic-content.meta/mc-programmatic-content/ssjs_triggeredSendTrackingClicksRetrieve.htm)
  - [Retrieve](https://developer.salesforce.com/docs/atlas.en-us.noversion.mc-programmatic-content.meta/mc-programmatic-content/ssjs_triggeredSendTrackingRetrieve.htm)
  - [TotalByInterval.Retrieve](https://developer.salesforce.com/docs/atlas.en-us.noversion.mc-programmatic-content.meta/mc-programmatic-content/ssjs_triggeredSendTrackingTotalByIntervalRetrieve.htm)
- SSJS Send Tracking
  - [Tracking.ClicksRetrieve](https://developer.salesforce.com/docs/atlas.en-us.noversion.mc-programmatic-content.meta/mc-programmatic-content/ssjs_sendTrackingClicksRetrieve.htm)
  - [Tracking.Retrieve](https://developer.salesforce.com/docs/atlas.en-us.noversion.mc-programmatic-content.meta/mc-programmatic-content/ssjs_sendTrackingRetrieve.htm)
  - [Tracking.TotalByIntervalRetrieve](https://developer.salesforce.com/docs/atlas.en-us.noversion.mc-programmatic-content.meta/mc-programmatic-content/ssjs_sendTrackingTotalByIntervalRetrieve.htm)
- [SSJS Stringify](https://developer.salesforce.com/docs/atlas.en-us.mc-programmatic-content.meta/mc-programmatic-content/ssjs_platformUtilityStringify.htm)
