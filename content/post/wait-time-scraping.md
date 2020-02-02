---
title: "Wait Time Scraping"
date: 2020-02-02T12:53:17-05:00
---

We recently had to visit the children's hospital here in Ottawa. We were lucky and were able to go early on a Sunday morning and didn't have to wait too long to be seen and treated. It made me curious what the wait time would have been had we not gone first thing in the morning. It also made me curious when the best time to go to the hospital would be as well as how wait times change of the week.

# TL;DR

*I created a wait time scraper for the children's hospital in town which auto updates a Google Sheet every 15 minutes.*

*See it [here!](https://docs.google.com/spreadsheets/d/1sfc95-dpw7GTgL71zygkJWoaA7JjNYmpKnPQIGHStvA/edit?usp=sharing) Please feel free to save your own version or copy the data elsewhere to play with!*

*Here are the graphs it's generating:*

![graphs](/images/graphs.png)

&nbsp;

---------

# How I Did It

The hospital has a [new wait time portal](https://www.cheo.on.ca/en/visiting-cheo/wait-times.aspx) that shows how many people are waiting and the longest time a person has been waiting.

![CHEO wait times](/images/waittime.png)

I decided I would scrape the time every 15 minutes and graph it so I could see the data over time.

First thing was to figure out how the webpage got the wait times. I was hoping they would be making some kind of API call instead of rendering the times server side, this would make parsing the time much easier.

Poking around the network tab in the browser dev tools I found a call to [https://www.cheo.on.ca/Common/Services/GetWaitTimes.ashx?&lang=en](https://www.cheo.on.ca/Common/Services/GetWaitTimes.ashx?&lang=en), there it was!

![network tab](/images/getwaittime.png)

The response from the server was nice JSON

```js
{
  "aveWaitMin": 77.44,
  "patientCount": 36,
  "longestWaitMin": 162,
  "lastUpdated":"1/1/1970 8:55:25 AM"
}
```

A neat thing is that there's a field they don't show on the webpage `aveWaitMin` which I assume is the average wait time, not sure how they're calculating it but still cool!

Next I created a script that would create a CSV line from the data:

```sh
#! /bin/bash
# gen-cheo-line.sh

export TZ=":US/Eastern"
export DATE=$(date '+%m/%d/%y %H:%M:%S')

export TIMES=$(curl -s 'https://www.cheo.on.ca/Common/Services/GetWaitTimes.ashx?&lang=en' | jq -r '[.patientCount, .aveWaitMin, .longestWaitMin] | @csv')

echo ${DATE},${TIMES}
```

*If you don't know about [jq it's definitely worth checking out](https://stedolan.github.io/jq/). It's a cool command line tool for manipulating JSON*

Now that I had a script to generate a CSV line I needed a way to run it every 15 minutes so I could grab the latest data. I knew cron jobs were a way to run periodic tasks and after some research a setup my own.

Run `crontab -e` to add or edit a cron job. 

I added the following job:

```sh
*/15 * * * * /home/me/gen-cheo-line.sh >> /home/me/cheo.csv
```

Now every 15 minutes (on the 15s) I would scrape the wait times and append them to a csv file that I could import to Google Sheets or Excel.

After spending a couple days coping and pasting the data to a spreadsheet I knew I needed to do something to auto update my sheet instead.

To do this I needed to figure out an automated way to update Google Sheets. I had hoped that they would have personal API keys but I couldn't find one. Instead I had to register an app for API access. 

Below are the instructions to obtain the credentials needed...

**I would recommend creating a new Google account so that you don't accidentally grant access to anything you don't mean to. This is what I did.**

&nbsp;

- Go to the [Google developers console](https://console.developers.google.com)
- Create a new project
![new project](/images/newproject.png)
- Enable the Sheets API
![enable sheets](/images/enablesheets.png)
- First you need to configure the consent screen for the app, create an external consent screen, add the `../auth/spreadsheets ` scope
- Next create crendentials, you need to use OAuth Client ID
![create credentials](/images/createcreds.png)
- For the type select "Other"
![create oauth client](/images/createoauth.png)
- Now you have a client ID and secret which will allow you to get access and refresh tokens to act as a particular user.
![oauth credentials](/images/oauthcreds.png)
- Now you need to get a token for your user. To get this you'll need to complete an OAuth flow. To do this I adapted the sample code from Google's Auth API and checked it in here https://github.com/danedmunds/google-oauth
- Download the client credentials and save them as `oauth2.keys.json` in the project
![download credentials](/images/downloadcreds.png)
- Run the project (`node index.js`) and complete login. **Beware what Google account you login as, if you want to use a throw away make sure you do here**. Once login is completed the project will dump the tokens to the console.
```
> node index.js
Code is <<AUTHCODEHERE>>
{ access_token: '<<ACCESSTOKENHERE>>',
  refresh_token: '<<REFRESHTOKENHERE>>',
  scope: 'https://www.googleapis.com/auth/spreadsheets',
  token_type: 'Bearer',
  expiry_date: 1580682770980 }
Tokens acquired.
{ expiry_date: 1580682770088,
  scopes: [ 'https://www.googleapis.com/auth/spreadsheets' ],
  azp: '<<AZPHERE>>',
  aud: '<<AUDHERE>>',
  exp: '1580682771',
  access_type: 'offline' }
```
- The important thing to capture here is the refresh token, this will allow scripts to obtain a new access token to act as the user at a later date.

Now I had the ability to obtain an access token for myself with a script...

```sh
#! /bin/bash
# token.sh

CLIENT_ID=CLIENTIDHERE
CLIENT_SECRET=CLIENTSECRETHERE
REFRESH_TOKEN=REFRESHTOKENHERE

curl \
  --data client_id=$CLIENT_ID \
  --data client_secret=$CLIENT_SECRET \
  --data grant_type=refresh_token \
  --data refresh_token=$REFRESH_TOKEN \
  https://oauth2.googleapis.com/token
```

Next I figured out how to append to a sheet with a script. [The docs for the API used can be found here](https://developers.google.com/sheets/api/reference/rest/v4/spreadsheets.values/append).

After some fiddling around I managed to create a script to get the wait times and append to the spreadsheet!

```sh
#! /bin/bash
# sheet.sh

ACCESS_TOKEN=$(./token.sh | jq .access_token)
DATE=$(TZ=":US/Eastern" date '+%m/%d/%y %H:%M:%S')
TIMES=$(curl -s 'https://www.cheo.on.ca/Common/Services/GetWaitTimes.ashx?&lang=en' | jq -r "[\"$DATE\", .patientCount, .aveWaitMin, .longestWaitMin]")

curl 'https://content-sheets.googleapis.com/v4/spreadsheets/<<SHEETID>>/values/A1%3AD1:append?includeValuesInResponse=true&insertDataOption=INSERT_ROWS&valueInputOption=RAW&alt=json' \
  -H 'Accept: */*' --compressed \
  -H 'Authorization: Bearer '$ACCESS_TOKEN \
  -H 'Content-Type: application/json' \
 --data "{\"values\":[${TIMES}]}"
```

The very last thing was to setup the cron job to run the script every 15 minutes:

```sh
*/15 * * * * /home/me/sheet.sh
```

TADA! Now I have an automated scraper that appends to my google sheet. I created some graphs on the sheet that pick up new values as well.

[See the sheet here.](https://docs.google.com/spreadsheets/d/1sfc95-dpw7GTgL71zygkJWoaA7JjNYmpKnPQIGHStvA/edit?usp=sharing) Please feel free to save your own version or copy the data elsewhere to play with!

Here are some graphs the sheet is generating

![graphs](/images/graphs.png)

So when is the best time to go to the children's hospital? Around 8:30AM ;)