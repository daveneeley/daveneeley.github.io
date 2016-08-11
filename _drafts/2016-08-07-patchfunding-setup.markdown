---
layout: post
title: Patchfunding setup
categories: scouts
date: 2016-08-11 03:00
---
#Setup
This post covers first-time setup of records on patchfunding.com. Once you have every address imported, it's fairly smooth sailing. The end result is to have an Excel workbook with several sheets:

 - A sheet containing all addresses in your unit from county records
 - A sheet containing all addresses currently in patchfunding
 - A sheet containing the merged set of the above, organized by routes

## Get a Map
For this to work, you need a map or list with all the street addresses in your unit. Some people suggest, particularly in LDS units, that you can use the membership records and import those. BUT, those records do not cover every address in the unit, the addresses follow no standard, there are frequently duplicate records with different homeowner names (different households at the same address or old records that need to be moved out of the unit), etc. Just don't use them.

My ward has a printed map that I assume came from [this company](https://www.facebook.com/UTAH-Mapping-445791522151116/?ref=br_rs). It shows every street number and street name in our unit. This is important, because you must know the street name and address range you want in order to download from the online county records.

## Get the County Records
The goal of this step is to have a single spreadsheet with a normalized set of addresses and property owners. 'Normalized' means the street names are abbreviations are consistently applied, spelling errors are fixed, no extra punctuation is included, etc. The records I downloaded from the county were already normalized. I went to the [Assessor's homepage](https://slco.org/assessor/), and selected 'Interactive Parcel Map'. I searched for each street name from my printed map, and then exported the records that were within the address range from each street. Our unit only had 10 or 15 streets in it, so I ended up with that many individual comma-separated-value (csv) files. I compiled the records from each csv file into a single spreadsheet. Name the sheet 'county'.

## Add the Patchfunding records on another sheet
Get the list of donors in your unit from the All Donors page. After logging in, click Donors -> All Donors to get there. Then select '5k Records per Page' to view the whole list. Then select all the rows in the table that is shown. Copy and paste those into another sheet in the same Excel workbook. Name the sheet 'patchfunding'.

I should note that you can try skipping this and the next two sections, and just rely on patchfunding's resolve duplicates functionality to cover the duplicate imports. On the other hand, if you want to get a good picture of route-by-route progress, these steps may still be useful to you.

## Match up the county records and the patchfunding records
Flipping back and forth between the 'county' and 'patchfunding' sheets, it should be easy to see the difference in address styles. You need to format them the same way for matching purposes. Since my county records were already normalized, I updated my patchfunding records to use the exactly the same formatting. I fixed mispelled street names and other common errors. This was pretty time consuming, because there's a million ways to get an address wrong...just ask the pizza delivery guy!

To assist with matching, I chose to add several extra columns to each sheet, with formulas. The 'county' sheet had just two important columns, 'owner_name' and 'property_location', but for route sorting a few more columns can help. I ended up with 9 additional columns. 'StreetNumberIsOdd', 'StreetNumber', 'StreetDirection', 'StreetName', 'StreetKey', 'Owner1LastName', 'Owner1FirstName', 'Owner2LastName', 'Owner2FirstName'. The purpose of these should be pretty clear.

## Organize the combined records into routes
The most important extra column is 'StreetKey'. I wanted to avoid the problems with misspelled or missing street appelations, like Court or CT, Way or Wy, Lane, etc. Missing directionals were also important to avoid. So 'StreetKey' contains the value from 'StreetNumber', an underscore, and the first 6 characters of 'StreetName'. This was sufficient for my neck of the woods. All of our street numbers have 4 digits, and all the street names are somewhat unique. 

Sort the data on the 'county' sheet by 'StreetName', then by 'StreetNumber'. Then cut/paste any side streets into their geographical locations. Make a third sheet called 'routes'. Use the county data as your baseline by adding the values from the 'StreetKey' column on the 'county' sheet as column A on the 'routes' sheet. For example, if my 'StreetKey' was in column L on the 'county' sheet, and the first value was in cell 2, my formula on the 'routes' sheet in cell A2 would be '=county!L2'. Then just copy that formula down the row until all the keys are shown. (Hopefully you've guessed by now that I'm not an Excel guru!)

Then you merge or combine the data from each source sheet in the manner that makes the most sense for you. You'll end up with some crazy formulas like this one:

=IF(NOT(ISNA(INDEX(patchfunding!AF:AF,(MATCH($A2,patchfunding!$AC:$AC,0))))),INDEX(patchfunding!AF:AF,(MATCH($A2,patchfunding!$AC:$AC,0))),INDEX(county!M:M,(MATCH($A2,county!$L:$L,0))))

This says 'If the value from cell A2 has a match in column AC of the patchfunding sheet, display the value in column AF of the same row, otherwise use the value of column M from the matching row on the county sheet'. For my setup, this means use the 'Owner 1 Last Name' from patchfunding records if it exists, otherwise use 'Owner 1 Last Name' from the county records.

## Import the routes one at a time
