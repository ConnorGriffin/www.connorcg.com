---
layout: post
title: YNAB Weekly Spending Reports with Google Apps Script
date: 2018-09-04 00:00:00 -700
modified: 2018-09-08 00:00:00 -700
categories: YNAB
tags:
- Google Apps Script
- YNAB
- API
- Finance
comments: true
---

[You Need A Budget](https://ynab.com/referral/?ref=YK_lCy_BFqyQo6f0&utm_source=customer_referral) is an online subscription-based budgeting application that I've been using since mid-2015. It's incredibly helpful in maximizing savings and reducing the guilt in making large purchases, among countless other things. Reddit user *doctors_like_cash* was looking for a way to email a weekly summary of activity by category, so I whipped up an example using [Google Apps Script](https://developers.google.com/apps-script/), which allows you to excute code on a schedule without having to host a server or run anything locally.

![Example YNAB spending report](/assets/img/ynab-spending-report-2018-09-03_23-31.png){: .center-image }
<!--more-->

## Building the Report
You will need to have a Google/Gmail account for this to work, since Google Apps Script uses your Gmail account to send the emails. You will also need a YNAB account, of course. 

### Generate a YNAB Personal Access Token
First we have to get a Personal Access Token for YNAB. Check out YNAB's [Authentication Overview](https://api.youneedabudget.com/#authentication-overview) and follow the instructions to generate a Personal Access Token.

### Create a Google Apps Script Project
Google Apps Script lets us run Javascript on Google's servers for free, which is awesome! Create a new project by clicking [here](https://script.google.com/intro), or by clicking **Start Scripting** on the [Google Apps Script page](https://www.google.com/script/start/). I've named my project *YNAB Weekly Spending Report*. 

### Add The Report Code
Replace the contents of the default **Code.gs** with the following script. One of the things the report does is go into your Gmail inbox and mark the message as unread. This is necessary if you're sending to and from the same address, since Google will automatically mark the received message as read. If you're sending to a non-Gmail address you should remove lines 71-80.

Due to requests from various users on Reddit, I have provided a few alternative scripts that change the way the reports are specified and reported on. 
* [Standard Report (the version below)](https://gist.github.com/ConnorGriffin/bbae5d2a99a58aca847f413dd2540b73) - Child categories are specified, the report shows totals for each child category.
* [Category Group Detail](https://gist.github.com/ConnorGriffin/d08a73eb3f679c7868122708e3a1f837) - Master categories are specified, the report shows totals for each child category.
* [Category Group Summary](https://gist.github.com/ConnorGriffin/7f85493d17e3eefd83a1e5bd88ce91d5) - Master categories are specified, the report shows totals for each master category.

Fill in these fields with your own details:  

`accessToken`: Your YNAB Personal Access Token  
`budgetName`: Name of the budget to use  
`categories`: Your desired category names to monitor, in Javascript array syntax (example provided)  
`recipient`: Email recipients, comma separated 

{% gist bbae5d2a99a58aca847f413dd2540b73 %}

### Run the Report
From the **Select function** dropdown, choose the **sendYnabReport** function, then press Run (the play button).

![Select function](/assets/img/select-function-2018-09-04_00-36.png)


You will need to authorize the project to do a few things:

* Access your Gmail to mark the sent email as unread, otherwise it just sit in your sent box
* Send the report email
* Connect to api.youneedabudget.com

![Permissions request](/assets/img/permissions-2018-09-04_00-43.png)

After authorizing the project, you should receive an email that looks like the example at the top of the page. 

### Schedule the Report
All of this is great, but now we have to schedule the script to run weekly. 

Click the timer icon and create a new trigger. I've setup a weekly trigger that runs every Monday between midnight and 1 AM.

![Project triggers](/assets/img/triggers-2018-09-04_00-56.png)

## You're Done!
And that's it, now you should get a report every week on your spending habits. 

I encourage you to dig through the code and look into adding more columns, links to categories, etc. The opportunities are endless.
