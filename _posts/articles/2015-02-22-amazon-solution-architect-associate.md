---
layout: post
title: "Passed Amazon Solution Architect - Associate Level Exam"
excerpt: "My experiences preparing for the exam, tips etc"
categories: articles
tags: [AWS, Exams]
comments: true
share: true
---


I am pretty excited that last week, I have successfully completed [Amazon Solution Architect - Associate Level](http://aws.amazon.com/certification/certified-solutions-architect-associate/) exam. Being part of a [wonderful team](http://www.minjar.com/) that designs, implements and maintains complex architectures day in and day out certainly helped me pass this exam. I am also indebted to CloudAcademy, particularly the courses on certification by David Clinton were very useful. Each course is full of useful quizzes and these are a lot tougher than the actual exam itself. They also have a mock certification exam with about 250 questions. Below are their courses:

1. [AWS Solutions Architect Associate Level Certification Course -  Part 1 of 3](https://cloudacademy.com/amazon-web-services/courses/aws-solutions-architect-associate-level-certification-course-part-1-of-3/)
2. [AWS Solutions Architect Associate Level Certification Course -  Part 2 of 3](https://cloudacademy.com/amazon-web-services/courses/aws-solutions-architect-associate-level-certification-course-part-2-of-3/) 
3. [AWS Solutions Architect Associate Level Certification Course -  Part 3 of 3](https://cloudacademy.com/amazon-web-services/courses/aws-solutions-architect-associate-level-certification-course:-part-3-of-3/)

I have completed all of the above courses and another course which talks about Amazon VPC. What really helped me were the explanations to each question with quotes and links from relevant parts of amazon documentation. When I answered a certain question incorrectly, I used to copy the relevant links and explanation to a document to read it again. After 3 days of marathon CloudAcademy, my dashboard looked like this:


<figure>
	<img src="/images/cloudcademy.png" alt="image">
</figure>


Here are a few tips that might help you prepare better for this exam. 

* Read all the white papers suggested by Amazon [in this document](http://awstrainingandcertification.s3.amazonaws.com/production/AWS_certified_solutions_architect_associate_blueprint.pdf)  
* Complete all the above courses. David Clinton shows how to build a highly available media wiki application and a wordpress site on above courses. He covers pretty much everything, right from designing your VPC with multiple subnets, configuring routing tables, creating multiple subnets etc  
* I was lucky that I had access to an amazon account to launch services and play around with. Even if you don't want one, you can pretty much test everything thoroughly at around 8$ (well except redshift clusters etc). So get an account, create VPCs, subnets, play around with routing tables, Network ACLs, build a highly available app and auto scale it etc.   
* Read [Amazon FAQs](http://aws.amazon.com/faqs/). They are very thorough and you might find answers to most of your future examination questions. Read EC2, VPC, S3/Glacier, IAM FAQs completely.  
* During the exam, read the question carefully. When asking questions about less popular services like SQS, amazon is nice enough to add a couple of completely irrelevant options to make things easier for you. So read the questions and multiple choices carefully and when you don't know the answer for sure, use the process of elimination. :)  

I am currently preparing for [Implementing Microsoft Azure Infrastructure Solutions](https://www.microsoft.com/learning/en-in/exam-70-533.aspx) and I hope to share my experiences once I a appear for the exam. Meanwhile, good luck and happy preparing! 