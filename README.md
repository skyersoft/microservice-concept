# Microservice Concepts

## How to separate a monolithic application's database

One of the most discussed topics in microservice world is 
how to design the tables which are related to each other. 
It is easy to design a single database application. 
Firstly, you can design with the maximum normalization and with the 
performance constraint/requirements it can be decreased for the specific 
fields.

Let's explain the concepts with a simplistic scenario. 
We have a web page to make quizzes for the customers. The table design is as following:

|Organization Table|
|------------|
|ID|
|Org name|
|Contact email|
|Token|


|User Table|
|-----------|
|ID|
|Organization_ID|
|Name|
|email|


|Questions Table|
|---------------|
|ID|
|Order|
|Desc|


|Question Options Table|
|------|
|ID|
|Question_ID|
|Desc|
|Points|


|Quiz|       
|----|       
|ID|
|Token|
|User_ID|
|start_date|
|end_date|
|status|

|Answers|
|------|
|ID|
|Quiz_ID|
|Question_ID|
|Options_ID|

Admin enters the organization name and contact email from that organization.
System sends an email to the contact with a link to enter the related page add workers of the organization.
With each entry workers get emails which include links to their own quiz links.
After the worker gets the email and opens the page quiz starts and shows the questions one by one.
With each answer the results are recorded and if all the questions completed quiz ends and worker cannot use the same link again.

Our first application is monolithic and all the tables are in the same database. However, when we want to calculate the 
 score for  the quiz we need to join the quiz & answers & options tables to get a simple score. Instead of this,
what we prefer is add a new *score* field to the *Quiz* table. So we can reach the score of the quiz without any calculation.
Since the answers of the quiz cannot be updated after the quiz, we don't need to update the quiz score.

### Converting the legacy application to microservices

There are many advantages of microservice architecture, but these advantages comes with a cost. We will concentrate on the cost side later.

#### Vertical Functionality 
We can understand the vertical functionality by understanding our user groups. We have four main user groups. 
* One of them prepares the questions and their options, scores, etc.
* The other one contacts with the companies and enters the company info to the system and tracks the completion of quizzes.
* The customer company manager populates the worker data and tracks the process
* Workers who take the quizzes

For the sake of simplicity result analysis is not included in the scenario.
Although it seems all the functionality are glued together and there is one quiz application we decided to functionality to at least four microservices.

#### Performance and Horizontal Scalability
Performance problem starts when all the workers get their emails, and they start the quiz. After the analysis we realised that
company POCes queries the results frequently to track if everything is going on the track. In addition to this, since every question
must be answered in 15 seconds, we get 1000+ requests every 15 seconds to get the question and another 1000+ to save the answer.
All the system slows down, and we can't serve to the other customers.
From the performance point of view we can divide the system into these pieces:
* Querying the next question
* Saving the answer for question
* Company wide querying the started and not started quizzes 
* Other a few users or rarely used parts

#### Data Encapsulation
In the old days when internet/network bandwidth was so slow and was going down frequently nearly all the banking system
was using distributed databases. Branches of the banks had their own databases and application servers. They were synching 
with the central bank when the network allows.

If you have multiple databases, in order to minimise the headache you must define the owner of the data. It is basically, who can insert/update/delete the data.
Querying is not a problem but depending on the queried data to do something or to update another data must be considered carefully.
If you can only deposit or withdraw money just from your branch, do we need to send your account information to other branches?
Even if we send the data, they won't be able to update it so we can just send account number and account owner name to other branches.
This data isn't updated and can be used by other branches to send money to this account.
So this concept is a different type of *Data Encapsulation* from the data ownership point of view. We can use the same concept in our scenario.
* Organization table is only updated by our salesperson, all other roles just read this
* User table only updated by organization POCes, all other roles just read this
* Questions and options tables are only updated by our supervisors
* Quiz table rows created by system with trigger of company POCs, but updated by the quiz takers.
* Answers are updated only by quiz takers.

So from this analysis our roles match with the data ownership, and we can divide the system into four microservices.

### Define the Core microservice
After the analysis of the system, we can easily say that our core system which doesn't need any other integration is question bank.
If we can first implement this microservice we can build the other services around core one. So what will be in this service.

|Questions Table|
|---------------|
|ID|
|Order|
|Desc|


|Question Options Table|
|------|
|ID|
|Question_ID|
|Desc|
|Points|

We have just two tables in this service. We can start the MVP microservice by providing just one API which brings the questions in order one by one.
We can populate the data from our spreadsheets since we already have the questions in our hand.

### Define the implementation order of microservices
From the process point of view next step is defining the organization which will take the quiz. Also, we can understand this from
the table since there is no FK in *Organization* table. It seems that creating a microservice just for one table is over-engineering.
On the other hand data ownership, vertical functionality and data encapsulation analysis say that we can divide it.
What we may gain if we divide it. If we want to add functionality for the salesperson such as keeping all the possible customers, their contacts,
payment information, etc. We don't need to share this data with the other services. Suddenly, this database became highly sensitive.

#### Sales Microservice
This service doesn't need any other data. Salesperson enters the organization and related information. System sends an email to the POC
with a token to access the system in order to enter the list of quiz takers.

|Organization Table|
|------------|
|ID|
|Org name|
|Contact email|
|invoice info|
|Invoice amount|

|POC|
|----|
|ID|
|Org_ID|
|name|
|email|
|phone|
|Token|

What have we changed? We added multiple POCes for an organization. They all have their own tokens, etc. Also, organization has invoice data assuming that it is one line.

#### POC Microservice 
After POCes get the emails, they have to enter the email and other information of quiz takers. This can be uploaded with a spreadsheet or csv file.
Which data will we have in this microservice?

|Organization Table|
|------------|
|ID|
|Org name|

|POC|
|---|
|ID|
|name|
|Organization_ID|
|token|


|User Table|
|-----------|
|ID|
|Organization_ID|
|Name|
|email|

API got the email and other information saved them to the database and sent that data to the message queue. Those messages will be taken by quiz microservice.

#### Quiz Microservice
Quiz microservice after getting the email addresses, it saves the related quiz rows and sends emails to the related people with proper token info.
Quiz takers gets the emails and uses their tokens to access the quiz. By clicking the start button, their clocks begin to tick.
Quiz microservice sends these messages:
* Quiz started
* Question answered
* Quiz completed

Any other microservice can listen these messages to show the real-time tracking information.

|Quiz|       
|----|       
|ID|
|Token|
|User_ID|
|start_date|
|end_date|
|status|

|Answers|
|------|
|ID|
|Quiz_ID|
|Question_ID|
|Options_ID|