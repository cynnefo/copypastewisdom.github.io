---
layout: post
title: Remove Specific Email Address From Multiple Rows
excerpt: TSQL fun with REPLACE
categories: articles
tags:
- Databases
- TSQL
date: '2014-09-28T11:49:17.000+00:00'
comments: true
share: true
---

One of our customers had this problem where they add the Customer Service Manager's address to all accounts they manage manually. When this guy leaves the company, someone has to use the UI and manually go through thousands of records one by one and remove the email id. The column which stores these email values in the table sometimes has multiple emails separated by a **comma(,)** and the specific email that needs to be removed could be at the beginning, end or any any other random place in the string. We wanted to create a stored procedure which accepts target email as parameter and deletes all occurrences. This whole thing appeared too complex until I figured out all it takes is an update command and use of TSQL function `REPLACE`. 

Lets say we would like to remove `joker@GothamCity.com`. Who in his right mind would want to keep the Joker :). Lets have some test data to play around. 

{% highlight sql %}
/* Create a test table */
CREATE TABLE EmailTest 
(
	id int identity,
    name nvarchar(100),
    email nvarchar(500)
   ) 
/* Insert some random data */ 
/*The following is not a valid record for our test */
  INSERT INTO EmailTest VALUES ('Nikola Tesla', 'tesla@cosmos.com')
/* 'Joker' is what we are going to update/remove. Lets insert a string with a lone email id. */
  INSERT INTO EmailTest VALUES ('Joker', 'joker@GothanCity')
/* What if the email id is at the beginning of the string? */
  INSERT INTO EmailTest VALUES ('JokerAndCo', 'joker@GothamCity.com,tesla@cosmos.com,vigilante@GothamCity.com,lannister@WestrOS.com,whitman@leavesofgrass.com')
/* Email at the end of the string */
  INSERT INTO EmailTest ('Alan Turing', 'turing@cryptoland.com,vigilante@GothamCity.com,lannister@WestrOS.com,tesla@cosmos.com,joker@GothamCity.com') 
/* Email at a random position */
  INSERT INTO EmailTest VALUES ('Elon Musk', 'emusk@spacex.com,tesla@cosmos.com,joker@GothamCity.com,Vigilante@GothamCity.com,Lannister@WestrOS.com')
{% endhighlight %}

The actual stored procedure. Its self explanatory.

{% highlight sql %}
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

CREATE PROCEDURE [dbo].[RemoveEmail]
	@RemEmail NVARCHAR(100)
AS

DECLARE @Length int
DECLARE @Delimiter char
SELECT @Delimiter = ','
SELECT @Length = LEN(@RemEmail)

UPDATE EmailTest SET email = REPLACE(email, @RemEmail, '') WHERE LEN(email) = @Length #lone record
UPDATE EmailTest SET email = REPLACE(email, @RemEmail+@Delimiter, '') WHERE CHARINDEX(@RemEmail, email) = 1 #beginning of string
UPDATE EmailTest SET email = REPLACE(email, @Delimiter+@RemEmail, '') WHERE CHARINDEX(@Delimiter, REVERSE(email)) - @Length  = 1 #end of the string
UPDATE emailtest SET email = REPLACE(email, @RemEmail+@Delimiter, '') #random position in the string 

{% endhighlight %}

Execute stored procedure

`EXECUTE dbo.RemoveEmail 'joker@GothamCity.com'`

Hope this is helpful to someone. Enjoy!
