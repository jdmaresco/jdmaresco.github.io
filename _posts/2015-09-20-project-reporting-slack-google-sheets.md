---
layout: post
title:  "Use Slack and Google Sheets to automate project reporting"
date:   2015-09-20 21:00:00
categories: tutorials
comments: true
---
{:center: style="text-align: center"}

### Introduction

If you're working for a small- to mid-sized company that was founded in the last 10 years, chances are you're using Google Apps everyday for mail, spreadsheets, documents, and, well... maybe your entire job. In this tutorial, we're going to use Google Apps Script, a powerful JavaScript-based programming language built in to all of your Google Apps. Chances are, you may not even realize how much power is available to you – you can create custom add-ons for your company, connect to outside services, and add new formulas to your spreadsheets. We can even do crazy things like send a Gmail from a custom menu in Google Sheets, or create a new Google Doc whenever someone submits a Google Form.

### What's Slack?

[Slack](slack.com), a business chat and productivity app, was introduced only 2 years ago, but has quickly taken off as one of the most-used productivity tools in the startup world today. It has many competitors in the "office chat" space (like Hipchat and Yammer) but none have achieved its scale.

### Prerequisites

- Understand how to use a little bit of JavaScript
- Have a Google or Google Apps account
- Have access to add integrations to a Slack team (you can [create a new team](https://slack.com/create) for free)

### Let's get started

**Our goal:**  To allow users to participate in an agile-style "stand up", by sending a message in any Slack channel.

![](/images/slack-google-sheets/01.png)
{:center}

These updates will be sent to a Google Spreadsheet.

![](/images/slack-google-sheets/02.png)
{:center}

Finally, a message will be posted back into another channel, summarizing the update.

![](/images/slack-google-sheets/03.png)
{:center}

The benefit of our chosen workflow is that teams can post their stand-ups into their own project team channels, while the **#updates** channel can host a running log of all updates.

### Set up your spreadsheet

First we'll create a new spreadsheet and set up the rows that will hold our data. Let's freeze the top row for convenience, and then name the columns as shown below.

To make these columns easier to interact with, we're going to make use of **Named Ranges**. This allows us to refer to a group of cells by an arbitrary name, rather than their cell reference. To name a range, select the column, then choose **Data** > **Named Ranges**. Give it a name (I prefer one-word lowercase) and hit **Done**.

Go ahead and name all 6 of your new columns.

![](/images/slack-google-sheets/04.png)
{:center}

### Create a script

Now comes the fun part. From the **Tools** menu, choose **Script Editor**. You'll see a pop-up menu with lots of helpful links, but for now, click **Blank Project**.

Erase the existing code in the editor and add the following code.

{% highlight javascript %}
function doPost(request) {
  var sheets = SpreadsheetApp.openById('[YOUR SPREADSHEET ID]');
  var params = request.parameters;

  // MORE CODE WILL GO HERE
}
{% endhighlight %}

Let's look at the first 3 lines of code:

1. **We create a `doPost(request)` function so that our script can receive data via POST requests.** This is a special function that will be called on to process any new POST request, once we've deployed our script as a web app (which we will do momentarily).

1. **We use the SpreadsheetApp API to get access to a spreadsheet file.**  Google Apps Script differentiates between a **spreadsheet** and a **sheet**. In short, a **spreadsheet** is a file that may have one or more **sheets** (a.k.a. worksheets or tabs). Since we're interacting with our spreadsheet via **named ranges**, we just need to get a reference to the spreadsheet file. We do this by referencing the ID (which is the long string of characters in the spreadsheet's URL).

    **Replace the placeholder in the code with your spreadsheet ID.**

1. **The `request.parameters` object will contain the entire payload / data of the incoming POST request.** We assign it to a variable `params` for convenience, and we'll make use of this later.

*Note:  Not all Google Apps Scripts have to be "attached" to a Google Sheets file, but the one you've just created is tied directly to your spreadsheet. If you close the spreadsheet, your Script Editor tab will also close automatically. To re-open it, just follow the steps above to open back up your code.*

From the **Publish** menu, choose **Deploy as Web App**. If you haven't saved/named the project yet, you'll be prompted to do so. Give it a name and then click **OK**. Choose the following options:

- Execute the app as: **Me**
- Who has access to the app: **Anyone, even anonymous**

![](/images/slack-google-sheets/05.png)
{:center}

Click **Deploy**, then copy the address of the **Current web app URL** to your clipboard. You'll need that in the next step.

<br>

### Create an Outgoing WebHook to send information to Google Sheets

In order to get data out of Slack, we create a **WebHook**.

Go to **https://[your Slack team].slack.com/services/new** and find **Outgoing WebHooks**.

![](/images/slack-google-sheets/06.png)
{:center}

Click **View**, then **Add Outgoing WebHooks Integration**.

In the next screen, take a look at the section titled "Outgoing Data". This is a preview of the data that our script will receive when someone sends an update. For example, if someone posts the update from a Slack channel called **#myProject**, we could access the name of that channel in our script by using `request.parameters.channel_name`. Pretty simple.

Scroll down to the section titled **Integration Settings**. Choose the following settings:

- Channel: **Any**
- Trigger Word(s): **update:**
- URL(s): *Paste in the Apps Script URL you copied in the previous step*
- Token: *Copy the value in this field to your clipboard*
- Descriptive Label: **Stand-up Bot**

Ignore the rest of the fields, and click **Save Settings**.

<br>

### Record updates in the spreadsheet

Go back to your Google Apps Script Editor and fill out your `doPost` function with the following code.

{% highlight javascript %}
function doPost(req) {
  var sheets = SpreadsheetApp.openById('[SPREADSHEET ID]');
  var params = req.parameters;

  var nR = getNextRow(sheets) + 1;

  if (params.token == "[OUTGOING WEBHOOK TOKEN]") {

    // PROCESS TEXT FROM MESSAGE
    var textRaw = String(params.text).replace(/^\s*update\s*:*\s*/gi,'');
    var text = textRaw.split(/\s*;\s*/g);

    // FALL BACK TO DEFAULT TEXT IF NO UPDATE PROVIDED
    var project   = text[0] || "No Project Specified";
    var yesterday = text[1] || "No update provided";
    var today     = text[2] || "No update provided";
    var blockers  = text[3] || "No update provided";

    // RECORD TIMESTAMP AND USER NAME IN SPREADSHEET
    sheets.getRangeByName('timestamp').getCell(nR,1).setValue(new Date());
    sheets.getRangeByName('user').getCell(nR,1).setValue(params.user_name);

    // RECORD UPDATE INFORMATION INTO SPREADSHEET
    sheets.getRangeByName('project').getCell(nR,1).setValue(project);
    sheets.getRangeByName('yesterday').getCell(nR,1).setValue(yesterday);
    sheets.getRangeByName('today').getCell(nR,1).setValue(today);
    sheets.getRangeByName('blockers').getCell(nR,1).setValue(blockers);

    var channel = "updates";

    postResponse(channel,params.channel_name,project,params.user_name,yesterday,today,blockers);

  } else {
    return;
  }
}

function getNextRow(sheets) {
  var timestamps = sheets.getRangeByName("timestamp").getValues();
  for (i in timestamps) {
    if(timestamps[i][0] == "") {
      return Number(i);
      break;
    }
  }
}
{% endhighlight %}

Phew. That's a lot. Let's walk through the code. First we call a custom function called `getNextRow()` (we declare this at the bottom of the code block above). This function takes the values in the **timestamp** column in our spreadsheet, and loops through those values until it finds one that's blank. Then it returns a number corresponding to that index, so we know where to store the next set of values that gets submitted.

{% highlight javascript %}
function doPost(req) {
  ...
  var nR = getNextRow(sheets) + 1;
  ...
}

function getNextRow(sheets) {
  var timestamps = sheets.getRangeByName("timestamp").getValues();
  for (i in timestamps) {
    if(timestamps[i][0] == "") {
      return Number(i);
      break;
    }
  }
}
{% endhighlight %}


Note that the `getValues()` function that is provided by the Google API returns a two-dimensional array (`timestamps[row][col]`). Since we're dealing with a single column, we use index `0` to access to the one column in the range.

Once `getNextRow()` returns a value to our `doPost()` function, we add `1` to the result. The reason is that, a few lines down, we're going to use it in the `getCell(row,col)` function, which references rows and columns starting with an index of `1` (not `0` like arrays).

The rest of the body of our function is wrapped in an `if` block that checks to make sure the token in the POST request matches the one in our Slack WebHook configuration (so we can confirm no one is sending phony updates to our endpoint). Replace `[OUTGOING WEBHOOK TOKEN]` in the code above with the actual token you copied when creating the WebHook in Slack.

{% highlight javascript %}
function doPost(request) {
  ...
  if (params.token == "[OUTGOING WEBHOOK TOKEN]") { }
  ...
}
{% endhighlight %}

Next, we do some processing of the **text** parameter, which is just the raw text that is entered by the user in Slack. We use [Regular Expressions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Regular_Expressions) to:

- Get rid of the word "update:" at the beginning of the text
- Split the remainder of the **text** string on semicolons

We try to catch some common errors – it's possible that the user didn't enter in a full update, or maybe they forgot to add a semicolon, so we provide some fallback values, and then we record the update values into the spreadsheet.

The last row is commented out. We'll come back to this at the end to send a message *back* to Slack using an **Incoming WebHook**.

{% highlight javascript %}
// postResponse(channel, params.channel_name, project, params.user_name, yesterday, today, blockers);
{% endhighlight %}

Before we implement that function, let's test out our app. One quirk of Google Apps Script is that saving our file doesn't automatically redeploy it so we need to do that manually.

- **Publish** > **Deploy as web app**
- Project Version: **New**
- Click **Update**

_It's important that you change **Project Version** to "New" every time you redeploy. Thankfully, the URL does not change when you redeploy, so you don't have to change any settings in Slack._

<br>

### Test it out

Go into a channel in your slack group and type the following:

`Update:  Project X; Yesterday I got a lot done.; Today I will get more done.; I don't have any blockers.`

If all is working as expected, your spreadsheet should live-update within seconds.

<br>

### Create an Incoming WebHook to post back to Slack

We've enabled anyone to submit an update from their project channel. But we still want to have a summary of these updates posted back into a shared **#updates** channel so everyone can stay in the loop.

Go to **https://[your Slack team].slack.com/services/new** and find **Incoming WebHooks**.

Click **View**, then choose a channel (e.g. **#updates**). Click **Add Incoming WebHooks Integration**.

Scroll down to the section titled **Integration Settings**. Choose the following settings:

- Post to Channel: **#updates**
- Webhook URL: *Leave as default, click __Copy URL__*
- Descriptive Label: **Stand-up post to updates channel**

Leave the rest of the settings in their default state, and click **Save Settings**.

<br>

### Send a response from Google Apps Script

Return to the Google Apps Script Editor. We're going to add another function. Let's create a second `.gs` file by selecting **File** > **New** > **Script File**. Give the file the name `PostResponse.gs` and click **OK**.

Erase the existing code in the editor and add the following code.

{% highlight javascript %}
function postResponse(channel, srcChannel, project, userName, yesterday, today, blockers) {

  var payload = {
    "channel": "#" + channel,
    "username": "New Update",
    "icon_emoji": ":white_check_mark:",
    "link_names": 1,
    "attachments":[
       {
          "fallback": "This is an update from a Slackbot integrated into your organization. Your client chose not to show the attachment.",
          "pretext": "*" + project + "* posted an update for stand-up. (Posted by @" + userName + " in #" + srcChannel + ")",
          "mrkdwn_in": ["pretext"],
          "color": "#D00000",
          "fields":[
             {
                "title":"Yesterday",
                "value": yesterday,
                "short":false
             },
             {
                "title":"Today",
                "value": today,
                "short":false
             },
             {
                "title":"Blockers",
                "value": blockers,
                "short": false
             }
          ]
       }
    ]
  };

  var url = '[YOUR INCOMING WEBHOOK URL]';
  var options = {
    'method': 'post',
    'payload': JSON.stringify(payload)
  };

  var response = UrlFetchApp.fetch(url,options);
}
{% endhighlight %}

Replace `[YOUR INCOMING WEBHOOK URL]` with the URL that you copied to the clipboard when creating the Incoming Slack WebHook. This is the URL your script will post back to in order to send a message to Slack.

It should be fairly obvious what each of the parameters to this `postResponse()` function are:

- `channel`: The channel we're posting back to
- `srcChannel`: The channel the user originally posted their update from
- `project`: The name of the project specified in the update
- `userName`: The Slack username of the person who posted the update
- `yesterday`, `today`, `blockers`: The three components of the stand-up update

A few other things about the payload we're sending over to Slack:

- we can override some of the defaults that were specified when we created the **Incoming WebHook** (for example which Username or Icon Emoji are attached to the message)
- the `link_names` property in the payload tells Slack to turn references to @users or #channels into links
- the `mrkdwn_in` property in the `attachments` array tells Slack to parse markdown in the `pretext` field

You can read more about how Slack processes and parses attachments in the [Slack documentation](https://api.slack.com/docs/attachments).

The final line in this function takes care of actually sending the data over to the Slack Webhook.

{% highlight javascript %}
var response = UrlFetchApp.fetch(url,options);
{% endhighlight %}

One last step... go uncomment out the line in your `doPost()` function.

{% highlight javascript %}
postResponse(channel, params.channel_name, project, params.user_name, yesterday, today, blockers);
{% endhighlight %}

Try it out. You should see the data both register in your spreadsheet and post back to the **#updates** channel in Slack!
<br><br>
