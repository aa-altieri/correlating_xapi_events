# Project: Correlating Usage with xAPI

## Assumptions
This article assumes you have some familiarity with the [Experience API](https://github.com/adlnet/xAPI-Spec).  You don't need to know a LOT about it, but you should at least understand the basic statement structure (actor-verb-object), and have a basic understanding of how xAPI can be used in general.  I will do my best to explain each of the parts used here, as I go along.

I also assume you are ok looking at code segments.  Again, you need not be a Javascript expert or even much more than a beginner.  I will try to explain everything clearly as we walk through the examples.  However, it is not the goal or purpose of this article to teach the reader Javascript.  


## About this Project
Web content take a lot of work to create.  Especially video content.  You want to make sure that the videos you create accomplish their goals.  Do they convey the information you want them to?  This project is an example of how you can use xAPI to correlate video consumption with exam test answers.

The goal in this example is to show if the user played the part of the video where the answer to that question is given.  Then, we can show if watching that part of the video contributes to the success (or failure) in answering the exam question.  If most people answer the question incorrectly, and most of those people watched that part of the video, then it's likely not accomplishing the task of conveying that information.  

I presented on this topic at the eLearning Guild's DevLearn event in Las Vegas, on November 16th.  For that presentation I needed to show the following steps:

1. Show how to collect video consumption using xAPI
2. Show how to collect test answers using xAPI
3. Show how to issue a query for the video and test answer, and tie them together

As always, the technical machinations are far easier than the data management.  I had to consider these points in how I constructed the data to show that the person who answered this question also watched THAT part of the video:

1. How do I show that the user played a given SEGMENT of video?
2. How do I record where in the video the answer for that question is given?
3. How do I record this information in an xAPI statement in a way that's meaningful AND can be queried?
4. How do I build the queries to pull the information I need in a way that is meaningful?

In this article, I won't get into the data management strategies, apart from explaining the decisions I made and how they manifested themselves in my code.

### Tracking video
The first thing we need to do, is to begin capturing some experiences!  In this example, we are first going to track is someone played a video.
