---
title: "Energylog"
date: 2024-03-04T15:00:09Z
tags: []
featured_image: ""
description: ""
draft: true
---


# Displaying a log file (text file) in a Home Assistant Lovelace card

This uses the standard Lovelace Webpage (iFrame) card to display text from a file. A simple webpage uses JScript to create a table with text file lines displayed in each row of the table. The web page refreshed every 60 seconds to update the displayed content.

This example creates and displays a Home Assistant Notification log file with time-stamps, but any text file can be displayed with some minor modifications.



## Step 1 - Create a Home Assistant Notify file

This yaml should be included in the Home Assistant configuration.yaml file to create the log file that will be displayed. In this case we are using the log file to record automation events and the log file entries will include a timestamp (this is processed by the JScript below).

```
# Create log files for energy automation events
notify:
  - name: energy_log
    platform: file
    filename: /config/www/energylog.txt
    timestamp: true
```


## Step 2 - Add entries to the log file using the Home Assistant Notify service

The following (action:) code can be added to a Home Assistant automation to add an event to logfile

```
action:
  - service: notify.energy_log
    data:
      message: This is a sample log file entry
```


## Step 3 - Create a Lovelace Webpage card

Use a Home Assistant Lovelace Web page card (type: iframe) that will display the content of an html file (created below)

```
type: iframe
url: /local/energylog.html
aspect_ratio: 100%
title: Energy Log File
```


## Step 4 - Create html to display in the web page card.

The following html uses some straightforward JScript to read a text file (in this case a Home Assistant log file) and creates an html table.

### Notes on the code

The html meta/refresh reloads and updates the webpage every 60 seconds.

The file to read is specified as a URL parameter in the function call to the function "GetTextFileLines". In addition to the required text file name, the URL includes the current date and time ("/local/energylog.txt?" + Date.now()) - the date and time information are not used by the function but they ensure that each URL is unique. This prevents Home Assistant from caching the page.

The number of lines that will be displayed in the Lovelace card is determined by the "NumberOf Entries" parameter. This purely configures the number of table rows that are displayed. It does not affect the number of file lines that are actually read (the script will read the entire text file on each call, so may be slow with very large files).

The direction of output is determined by the "Direction" parameter.
```
<html>
    <meta http-equiv="refresh" content="60" />
    <head>
        <title>Log File</title>

        <style>

            table, th, td {
                font-family: Arial, sans-serif;
                font-size:14px;
                border-collapse: collapse;
                border-bottom: 1px solid black;
                border-top: 1px solid black;
                vertical-align: top;
            }
            td:first-child {
                white-space: nowrap;   /* Keep date and time together on a single line*/
                text-align: right;
                padding-right: 10px;
            }

        </style>

        <script>

            function FormatDatetime(DTString) {
                const DayNames = ["Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"];
                let DT = new Date(DTString);
                return `${DayNames[DT.getDay()]} ${DT.getDate()}, ` +
                    `${("0"+DT.getHours()).slice(-2)}:${("0"+DT.getMinutes()).slice(-2)}:${("0"+DT.getSeconds()).slice(-2)}`;
            }

            // Read the logfile and return entries in an HTML table
            // - Returns up to "NumberOfEntries" lines
            // - Entries are returned in order determined by "Direction" parameter

            function GetTextfileLines(FileUrl, NumberOfEntries, Direction) {

                let xmlHttp = new XMLHttpRequest();
                xmlHttp.open( "GET", FileUrl, false );
                xmlHttp.send( null );
                let Lines = xmlHttp.responseText.split("\n");

                let LineCount = Lines.length;
                let LastLineIdx = LineCount -2;   // Last entry is displayed first. The last line of the file is blank, so subtract 2 to get last line index
                let FirstLineIdx =  Math.max(2, LineCount-NumberOfEntries-1);  // ...count backwards but stop before the header lines

                let TableString = "<table>";
                // let TableString = "<table><tr><th>Date</th><th>Event</th></tr>";   // Table with Header Row

                if (Direction > 0) {
                    for (let i = FirstLineIdx; i <= LastLineIdx; i++) {
                        let SplitLine = Lines[i].split(/(?<=^\S+)\s/);       // Split on first space
                        TableString += "<tr><td>" + FormatDatetime(SplitLine[0]) + "</td><td>" + SplitLine[1] + "</td></tr>";
                    }
                } else {
                    for (let i = LastLineIdx; i >= FirstLineIdx; i--) {
                        let SplitLine = Lines[i].split(/(?<=^\S+)\s/);       // Split on first space
                        TableString += "<tr><td>" + FormatDatetime(SplitLine[0]) + "</td><td>" + SplitLine[1] + "</td></tr>";
                    }
                }

                return TableString + "</table>";
            }

        </script>
    </head>

    <body>

        <script>

            // Read up to 10 most recent lines from the Energy logfile and display as a table
            document.write(GetTextfileLines("/local/energylog.txt?" + Date.now(), 14, 1));

        </script>

    </body>
</html>
```