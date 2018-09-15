---
layout: post
title: Tracking Home Value in YNAB Using Zillow and Google Apps Script
date: 2018-09-10 02:00:00 -700
modified: 2018-09-15 13:00:00 -700
categories: YNAB
tags:
- Google Apps Script
- Automation
- YNAB
- API
comments: true
---

I've seen a few requests for YNAB ([You Need A Budget](https://ynab.com/referral/?ref=YK_lCy_BFqyQo6f0&utm_source=customer_referral)) to support some more advanced net worth tracking, such as tracking home value automatically. [Personal Capital](https://personalcapital.com) supports this out-of-the-box, but I thought it'd be fun to add it onto YNAB as well. 
<!--more-->

For this project we'll be using Google Apps Script again, since it's easy to use and accessible to pretty much everyone. For the home data we'll be using Zestimate, since it's pretty much the only publically available home value estimation tool. 

#### Generate a YNAB Personal Access Token
Follow the [Getting Started](https://api.youneedabudget.com/#getting-started) instructions on YNAB's API documentation page to generate a **Personal Access Token**. 

#### Get a Zillow Web Services ID (ZWSID)
1. Create an account on [Zillow](https://www.zillow.com/user/Register.htm){:target="_blank"}, or sign into your existing account.
2. Follow the [Get Started](https://www.zillow.com/howto/api/APIOverview.htm){:target="_blank"} instructions to sign up for API access.
3. Choose only the **Property Details API** unless you plan to do anything else with your ZWSID that might require access to the other endpoints. 
4. You should receive an email with your ZWSID shortly after filling out the form.

![API Signup](/assets/img/zillow-api-signup_2018-09-09_23-28.png){:height="759.75px" width="734.25"}  
*API Signup Page*{:.image-caption}

![API Email](/assets/img/zillow-zwsid-email_2018-09-09_23-28.png)  
*Confirmation Email*{:.image-caption}

#### Create a Tracking Account in YNAB
If you do not already have an account in YNAB for tracking your home value, create one now.  

1. From your budget screen click **Add Account**. 
2. Choose **Unlinked**.
3. Select **Asset (e.g. Investment)** from the account type dropdown, and then give the account a name and a starting balance.

*The balance can be the current Zestimate value, or you can leave it blank, since this script will create an inflow for you on first-run.*

#### Create a New Google Apps Script Project
1. Go to [script.google.com](https://script.google.com/) and choose **New script**, I've called my project *YNAB Zestimate Automation*.
2. Replace the entirety of **Code.gs** with the following script and **fill in your own details on lines 2-6**:  

{% gist faffa47631f77cde5829718123c859f7 %}

#### Run the Report
From the **Select function** dropdown, choose the **updateYnabHomeZestimate** function, then press Run (the play button).

![Select function](/assets/img/select-function-2018-09-10_01-36.png)

You will need to authorize the project to access external services (YNAB API, Zillow API).

![Project Permissions](/assets/img/ynab-zestimate-permissions_2018-09-09_23-28.png)  

#### Check Your Balance
If everything ran successfully you should have a new transaction in your tracking account, and your account balance should match your current Zestimate. 

![Home value](/assets/img/home-value_2018-09-10_01-39.png)
*My zestimate is $177,081 but my account balance was $180,000 so a $2,919 outflow was added*{:.image-caption}

#### Schedule the Report to Run Automatically
Click the timer icon and create a new trigger. I'm not sure how often Zestimates change, but I think once a month should be a good baseline for something like this.

![Project triggers](/assets/img/triggers-2018-09-10_01-51.png)