+++
date = "2015-02-26T18:22:41-05:00"
draft = false
title = "My Private API"

+++

I work at at a private school that leverages [Veracross](http://www.veracross.com/), a student information system(SIS) for almost everything we do. Attendance, grades,  inventory and any other data relevant to the organization finds it's way into Veracross in one way or another.

There are several ways of interacting with Veracross, the most common being ESWeb, a web interface for querying and interacting with the data. Veracross, also has an API, but it's (1) read only and (2) many of the queries that are possible in ESWeb are not possible via the API. This summer, we transitioned all our students to new email accounts which required me to do the following:  

**1.** Get list of students from Veracross.  
**2.** Create new username for each student.  
**3.** Update Active Directory.  
**4.** Update the students' email field in Veracross.   

As any sysadmin would've done, I wrote a script that automates this process. Ruby has [two nice gems for capturing data off a webpage](http://readysteadycode.com/howto-scrape-websites-with-ruby-and-mechanize) which I ended up using in my script. I use the Mechanize gem for loggin in to ESWeb, clicking a few links to get a list of students and export that data, generate the usernames and then for each student, log back in and update the necessary field. The same steps that I would've done if I had to do everything manually. 

It looks something like this:
```ruby
query_result = form.submit(button)
link = query_result.link_with(text: ' Export')
query_page = link.click
link = query_page.link_with(text: 'Comma-Delimited Text File')
csv_page = link.click
link = csv_page.links.last
agent.get(link.attributes.first[1]).save savefile
```
While doing this I realized that I might want to use this code in the future to automate other things, so I rewrote my code to be able to export any query quickly and in a repeatable way. What allowed me to do this were 2 features in ESWeb.

<img src="/vc_exportandsave.png"  />

A query can be saved, which gives it a name and a numeric id. The ID allows me to recall the same query over and over by just adding that ID to a URL.
```
esweb.asp?WCI=Results&Query=139187
```
All I need to do to get a different query is to change the ID.
I can then click export and save that query as a CSV file and then process the data further. Now I have all sorts of useful queries to save the data. For example, by writing a query that prints out any new students or new hires in Veracross, and then checking those results every day, I can generate notifications without waiting for HR or Admissions to tell me someone new is at the school. 

You might be wondering, where does the API come in? 
I'm used to writing code that interacts with various APIs, which usualy return code in JSON, not CSV. Luckily, adding that part was as easy as adding ```csv_output.to_json``` in the script. Now I have my own private API. 

The full script is[ now on Github](https://github.com/groob/vquery) and you can use it too. To get the result of a query, all you have to do is run the script with the query ID as an argument.   
```ruby ./vquery.rb 139187```

Any ESWeb user can do that. 
If you want to save the results to a csv file, use vquery_csv.rb instead.    
```ruby ./vquery_csv 139187 > my_spreadsheet.csv``` 
