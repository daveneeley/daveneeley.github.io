---
layout: post
title: Patchfunding Route Printing
categories: scouts
date: 2016-08-16 03:00
---
If you've ever tried using the default "Print Donor Cards" function from the patchfunding unit dashboard, you've probably noticed that it generates a giant single pdf file. This would be nice, except that the cards are sorted alphabetically by last name, not by routes. This makes the pdf file very difficult to use. Fortunately, there's an easier way.

# Theory
My [last post]({% post_url 2016-08-11-patchfunding-setup %}) covered a complicated way for overachievers to ensure that every address in their unit is in the patchfunding software. The key to that setup, and to printing routes easily, is in the route name. Each route name needs to be unique enough that you can search by it from the 'All Donors' screen without accidentally including records from other routes. I did this by adding a numeric key to each route name, plus a delimiter. Here's an example.

Let's say I have three routes. Two of them are on First Street, and the third is on Second Street. So I start out with these route names:

    First Street A
    First Street B
    Second Street 

Now, I browse to the All Donors list in patchfunding, and change the filter to show all records on one page. Then I type First Street in the search box, and hit enter. I get back all records from both 'First Street A' and 'First Street B', **plus** any records that have an address on First Street but are part of a different route. However, if I name my routes like this:

    01 - First Street A
    02 - First Street B
    03 - Second Street

I can change my search to '01 -' and get back only the donors on '01 - First Street A'. The space and hyphen together make up a delimiter that won't be found on any standardized US Post Office address, and hopefully won't be in any donor's name either.

# Practice
## Edit Route Names
To begin we need to edit our route names so they each have a unique key. From the unit dashboard in patchfunding, click 'Route Manager'. To edit an existing route, just click the route name. This pops open a small 'Edit Route' dialog. You can number your routes in whatever order makes sense for you -- just make sure to use each number only once. Also, for routes 1-9, use a leading zero in the route number. This will prevent search conflicts with route numbers greater than 10.

## Print Routes Individually
Now browse to the 'All Donors' page ('Donors' -> 'All Donors' from the page header). Update the filter to show all 5000 records on one page. Now search for route 1 by entering '01 -' into the search box and hit enter. Make sure all records in the result are correct, and that all records are selected. Then from the dropdown, you can select 'Print Donor Cards for the selected contacts' or 'Print Control Sheets for the selected contacts'. I recommend doing both. When you click the 'Do It' button, the browser will automatically download a pdf file called either 'FOS_control_sheet' or 'FOS_donor_cards'. I recommend renaming these files to include the route number you searched for. Repeat these steps, searching for '02 -' through your last route number.

Once you've got all your routes downloaded as pdfs, you can print them however and wherever you would like. The donor cards only print three to a page, so you can burn through a lot of ink if you print them from home! I prefer to copy all those pdf files into Google Drive or a USB stick, then drive over to the church and print them from the clerk's office. Make sure to keep each route separate when you do print them.

Give yourself enough time to print AND cut all the donor cards before your kick off meeting! That's a mistake I made the first year. If you review the pdf files, you'll note that the donor cards are still sorted by name instead of address. Last year I had my awesome family over on Friday night. They helped me cut, sort, and prepare each route envelope so the boys could take off running on Saturday morning.

# Closing Remarks
The best part of all of this is you should only have to set it all up once. If you followed my [last post]({% post_url 2016-08-11-patchfunding-setup %}) and imported the complete list of addresses from somewhere, then every address will remain in patchfunding for years to come. Having each address in patchfunding means you won't have to spend time printing up a separate list or tracking sheet to hand out with each route. They can just use the donor cards.
