# Project: Correlating Usage with xAPI

## Assumptions
This article assumes you have some familiarity with the [Experience API](https://github.com/adlnet/xAPI-Spec).  You don't need to know a LOT about it, but you should at least understand the basic statement structure (actor-verb-object), and have a basic understanding of how xAPI can be used in general.  I will do my best to explain each of the parts used here, as I go along.

I also assume you are ok looking at code segments.  Again, you need not be a Javascript expert or even much more than a beginner.  I will try to explain everything clearly as we walk through the examples.  However, it is not the goal or purpose of this article to teach the reader Javascript.  


## About this Project
Web content take a lot of work to create.  Especially video content.  You want to make sure that the videos you create accomplish their goals.  Do they convey the information you want them to?  This project is an example of how you can use xAPI to correlate video consumption with exam test answers.

The goal in this example is to show if the user played the part of the video where the answer to that question is given.  Then, we can show if watching that part of the video contributes to the success (or failure) in answering the exam question.  If most people answer the question incorrectly, and most of those people watched that part of the video, then it's likely not accomplishing the task of conveying that information.  

I presented on this topic at the eLearning Guild's DevLearn event in Las Vegas, on November 16th, 2016.  For that presentation I needed to show the following steps:

1. Show how to collect video consumption using xAPI
2. Show how to collect test answers using xAPI
3. Show how to issue a query for the video and test answer, and tie them together

As always, the programming is far easier than the data management.  I had to consider several points in how I constructed the data to show that the person who answered THIS question also watched THAT part of the video:

1. How do I show that the user played a given SEGMENT of video?
2. How do I record where in the video the answer for that question is given?
3. How do I record this information in an xAPI statement in a way that's meaningful AND can be queried?
4. How do I build the queries to pull the information I need in a way that is meaningful?

In this article, I won't get into the data management strategies, apart from explaining the decisions I made and how they manifested themselves in my code.

### Tracking video
The first thing we need to do, is begin capturing some experiences!  In this example, we'll first begin to track when someone played the video.  The video is being played using the [JWplayer](https://www.jwplayer.com/).  This has some great methods to track interactions with the video.  We'll display the video with this code:
```
<div id="mediaplayer1">
  <script type="text/javascript">
      jwplayer('mediaplayer1').setup({		  		
        width: "480",
        aspectratio: "16:9",
        file: "./big_buck_bunny.mp4",
        autostart: false,
      });
  </script>
```
This will create the DIV on the page.  Then, the script runs and draws the player object.  It also give the player a name of "mediaplayer1" so you can reference it later.

Capturing the user interactions such as Play, Pause, and Search are easily done with the API's.  These are defined by the methods such as `jwplayer().onPlay`, `jwplayer().onPause`, and `jwplayer().OnSeek` respectively.  For example, if you wanted to send a console message when the user hits the play button, you could use code like this:
```
jwplayer().onPlay(function(event){
  console.log("Video started at timestamp: " + jwplayer().getPosition());
});
```
So getting the data is easily done.  But we need to consider what data we NEED.  In this case, we need to know if the user played a given segment of the video, but we don't yet know what that segment will be.  This will be defined by the question.  We could record each play, pause, and stop statement.  Then, on the reporting side, we sort the statements by when they were sent.  But there are problems with this.  It requires a lot of logic on the reporting side such as  comparing when play and pause statements are sent.  What if the user hits play twice and there is no matching pause? What if the user hits pause twice?  What if a statement doesn't get sent for whatever reason?

So, to simplify this, I took the approach of sending the segment data when the user pauses or completes the video.  This statement includes the video timestamp where the play button was pressed, and the timestamp when the ending event (pause, completion, or abandoning the page) occurs.  So the first step is to collect the timestamp when play was pressed and assign that to a variable.  A simple modification to the above code does that for us:
```
jwplayer().onPlay(function(event){
  playFrom = jwplayer().getPosition()
});
```

Then, when the pause button is pressed, we save the current position in the video and send the statement.  Then we'll build the xAPI statement.  Normally, you would use a function to do this bit.  However, for this example project, I've done things very linearly to make it easier to read:

```
jwplayer().onPause(function(event){
  // save the current position where the user hit pause
  var position = jwplayer().getPosition()

  // Build the xAPI statement to send to the LRS
  var pausestatement = {
        "actor": {
          "mbox": "mailto:" + firstname + "@devlearn16.com",
          "name": firstname,
          "objectType": "Agent"
        },
        "verb": {
          "id": "http://adlnet.gov/expapi/verbs/Play",
          "display": {"en-US": "Video Played"}
        },
        "object": {
          "id": "http://example.com/bigbuckbunnyvid.html",
          "definition": {
            "name": {"en-US": "Big Buck Bunny Video"},
            "description": {"en-US": "sample description"}
          },
          "objectType": "Activity"
        },
        "result": {
          "extensions":{
            "http://example.com/xapi/period_start"  : playFrom,
            "http://example.com/xapi/period_end"  	: position
          }
        }
    };
//  Send the statement to the LRS
    var result = ADL.XAPIWrapper.sendStatement(pausestatement);
}); //end onPause
```
The above code is fairly straight forward.  When the pause button is pressed, the onPause() event runs.  That sets the current timestamp of the video then sends that as an xAPI statement along with the timestamp when the Play button was pressed, which was previously saved in the "playFrom" variable.  The key piece to look at is the result section:
```
"result": {
  "extensions":{
    "http://example.com/xapi/period_start"  : playFrom,
    "http://example.com/xapi/period_end"  	: position
  }
```

The result section defines exactly what you'd imagine, the results of the activity.  In this case, the results are that the user played a segment of video.  There are no pre-existing constructs for this information as part of the Result object.  So we extend the results by defining two new "variables."  These are defined using a URL, ensuring that they we can use them across multiple activities.

And that's it!  We've now started to collect activity streams for video playback.  There are some more examples in the source code.  And many other comments for various other aspects of the code.  I invite you to take a look.  It's not difficult at all to add these statements, even if you're not terribly familiar with Javascript programming.

So, now that we have the video information.  We need to look at collecting a matching test question!


#### A Word on Extensions:
xAPI does kind of have "variables."  But they're not the normal kind of variable you see in programming.  In normal programming, such as Javascript, you define a variable with a name, such as "playFrom."  This variable is defined within the scope of a functions or an entire page.  But xAPI crosses entire systems.  Once I send the variable "playFrom" to the LRS, the LRS has no definition for it because the variable definition exists only on the sending page.  So, xAPI defines extensions, which can be thought of as variables for now, by using a URL.  This way, anyone who looks at the data can look at the URL and know what the significance of the data is.  In this example, they do not resolve to a real web page.  But in a production release, it's best practice that you have a page listed, defining what each of the extensions is, and what it is used for.

### Tracking Quiz Answers
Much like with tracking video interactions, tracking quiz answers requires some thought.  You need to know what you want to know!  What I mean is, you need to ask yourself some questions:
1. Do I want to know if the student answered the question correctly?
2. Do I want to know which answer the student chose?
3. Do I just want to know that the student answered the question, and nothing more?

I'm sure there are other concerns that I'm not thinking of.  But these are the most important questions when considering a normal test or quiz answer.  But we're not reporting on a normal quiz answer, here.  We want to correlate how the user answered against his or her interactions with a video.  So our considerations are slightly different:
1. Did the user answer the question?
2. What answer did the user choose?
3. Did the user watch the part of the video where the answer is presented?

These questions require a certain amount of data to be answered.  In turn, we need to know:
1. When the student answered the question.  The timestamp on the statement will tell us this.
2. Which answer was chosen.  We can record that in the results section of the statement.
3. Where in the video the answer is given.  We can use extensions for this!

So, the first thing we need is a quiz that sends xAPI statements.  You can find that in the Quiz.html file.  To create and track the question, we're using a simple form here:
```
<form onsubmit="submission()">
      <label class="question"> What is the first animal you see in the video?</label>
      <label><input type="radio" name="question1" ts="15.6"/> Flying Squirrel</label>
      <label><input type="radio" name="question1" ts="15.6"/> Bunny</label>
      <label><input type="radio" name="question1" ts="15.6"/> Bird</label>
      <label><input type="radio" name="question1" ts="15.6"/> Butterfly</label>
      <br />
      <input type="submit" />
</form>
```

On each answer, you'll see four attributes:

* Input type = "Radio" describes what kind of button to use
* name="question1" describes the name of the button
* ts="15.6" tells us the timestamp in the Big Buck Bunny video where the answer can be found.
* The label text ("Flying Squirrel" for example), which denotes the answer chosen.

This gives us the information we need for the report.  So, now to collect it in the LRS.  

Building the xAPI statement is done in much the same way as normal, with your Actor, Verb, and Object.  However, in this case, we'll make use of the Results section of the statement:

```
"result": {
     "response"	: answer,
     "extensions": {
       "http://example.com/xapi/location" : ts
       }
 },
```

In the above code, you can see we're using the Response attribute to send which answer was chosen.  Once again, we extend the statement, defining the extension "http://example.com/xapi/location" to carry the timestamp where the information was stored in the video.  This will be very useful in the next section.

The end result is a statement that looks like this:
```
{
    "id": "7ef193d4-5c0c-4e75-ad13-0a10f2089a2b",
    "actor": {
        "objectType": "Agent",
        "mbox": "mailto:bob@training2017.com",
        "name": "bob"
    },
    "verb": {
        "id": "http://adlnet.gov/expapi/verbs/answered",
        "display": {
            "en-US": "answered"
        }
    },
    "result": {
        "extensions": {
            "http://example.com/xapi/location": "15.6"
        },
        "response": "Bunny"
    },
    "timestamp": "2017-01-28T20:52:48.542Z",
    "stored": "2017-01-28T20:52:48.542Z",
    "version": "1.0.0",
    "object": {
        "id": "http://omnesLRS.com/xapi/quiz_tracker",
        "definition": {
            "name": {
                "en-US": "xAPI Video Quiz"
            },
            "description": {
                "en-US": "Correlating quiz answers to video consumption"
            }
        },
        "objectType": "Activity"
    }
}
```
If you look at the response section, you'll see that Bob chose "Bunny" as his answer, and we'll find the correct answer at 15.6 second into the video.  If you look down a little more, you'll see the timestamp, which tells us when this statement was received by the LRS.

So now we know who answered, when he or she answered, what the answer was, and where to find the correct answer in the video.  With this information in hand, it's time to build a report!

#### Taking it Further
This article is meant to describe the process to gather quiz information for a simple report comparing quiz performance to video interactions.  So it employs a fairly narrow scope.  In production environment, your quiz questions may be about numerous videos or other interactions.  So you may want to add another attribute to your quiz answers listing which video the answer is found in, for example.  Or, if you are reporting against other types of activities, you may need completely different data entirely.  



### Bringing it All Together
So we now have data on the video interaction and data on the Quiz interaction.  Let's review each of the statements:

Here is the statement sent when the user hit pause when watching the video:
```
{
    "id": "ae7a072e-89ec-4eba-825d-5f279c052bd9",
    "actor": {
        "objectType": "Agent",
        "mbox": "mailto:bob@training2017.com",
        "name": "bob"
    },
    "verb": {
        "id": "http://adlnet.gov/expapi/verbs/Play",
        "display": {
            "en-US": "Video Played"
        }
    },
    "result": {
        "extensions": {
            "http://example.com/xapi/period_end": 17.6,
            "http://example.com/xapi/period_start": 0
        }
    },
    "context": {
        "platform": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/55.0.2883.95 Safari/537.36"
    },
    "timestamp": "2017-01-28T20:52:38.927Z",
    "stored": "2017-01-28T20:52:38.927Z",
    "version": "1.0.0",
    "object": {
        "id": "http://example.com/bigbuckbunnyvid.html",
        "definition": {
            "name": {
                "en-US": "Big Buck Bunny Video"
            },
            "description": {
                "en-US": "sample description"
            }
        },
        "objectType": "Activity"
    }
}
```

And here is the statement sent when the quiz question was answered:
```
{
    "id": "7ef193d4-5c0c-4e75-ad13-0a10f2089a2b",
    "actor": {
        "objectType": "Agent",
        "mbox": "mailto:bob@training2017.com",
        "name": "bob"
    },
    "verb": {
        "id": "http://adlnet.gov/expapi/verbs/answered",
        "display": {
            "en-US": "answered"
        }
    },
    "result": {
        "extensions": {
            "http://example.com/xapi/location": "15.6"
        },
        "response": "Bunny"
    },
    "timestamp": "2017-01-28T20:52:48.542Z",
    "stored": "2017-01-28T20:52:48.542Z",
    "version": "1.0.0",
    "object": {
        "id": "http://omnesLRS.com/xapi/quiz_tracker",
        "definition": {
            "name": {
                "en-US": "xAPI Video Quiz"
            },
            "description": {
                "en-US": "Correlating quiz answers to video consumption"
            }
        },
        "objectType": "Activity"
    }
}
```
So, from the above, we know:

* Bob selected "Bunny" as the answer to the question
* the video gave the answer at 15.6 seconds in
* Bob watched the video from beginning to 17.6 seconds, so did watch the part of the video where the answer was given

But we want to show this in a report.  That requires us to present this data in a proper, unified format.  As expected, this is the most difficult and involved part of the process.

The first problem that makes this process so convoluted is the combination of the limits on queries in xAPI, and our use of extensions.  In xAPI, you can only really query on a few things:

* Agent (as either an Actor or Object)
* Verb ID used (The URL bit)
* Activity ID used (The URL bit)
* The timestamp of the statement

Now, with these, you can do quite a bit!  You can find out in how many activities Bob participated in a given timeframe.  You could query for all statements where the student "passed" a given course.  All very useful things to know!

The issue is that you cannot, for example, query for all those who passed with a score of 80 or lower.  You can't query for those who gave a specific answer on an exam.  And you cannot query for those who watched a specific part of a video.  At least, not directly, because this information is stored in the EXTENSIONS of the statements or is simply not a part of the defined query process in xAPI.  So, instead, you'd have to query for all those who passed the exam, then crawl through those statements and select the ones with scores below 80.  Or query for statements where someone played a video, then crawl through the resulting list of statements for those where the extensions tell you they played from a timestamp X lower than, and a timestamp Z greater than your point Y timestamp.  This, itself, isn't terribly difficult.  But the difficulty can be made exponentially greater with bad (or worse, no) data planning ahead of time!

The second problem is one of data modeling: In order to bring the two otherwise-disparate data points together, we need to start with one, and describe the second as a function of the first.  In this example, we're going to use the quiz as the starting data point, and define the second point as a function of that list, as you'll see in a moment.  So, first, we'll send an xAPI query for all statements where the verb "answered" and the object "http://omnesLRS.com/xapi/quiz_tracker" are used.  This will get us a list of all users answered the question.  The next step will be to see which, if any, of these users also watched the video.  There are two ways to do this:
1. Run two queries:  The first to find out who answered the question with the proper verb and activity.  And the second to find out who watched the video.  Then compare the two for overlap.
2. Run the first query for who answered the quiz question.  Then, for each user who did, send a query for THAT user, with the proper verb and activity.

For the purposes of this example, I'll choose the later option.  It's not the most efficient option, at all, but does demonstrate both queries, as well as how to build them systemically.

I'm using the ADL xAPI driver, which makes issuing queries pretty easy.  In fact, the actual query itself is only four lines of code:

```
var quizParams = ADL.XAPIWrapper.searchParams()
quizParams['verb'] = "http://adlnet.gov/expapi/verbs/answered";
quizParams['activity'] = "http://omnesLRS.com/xapi/quiz_tracker";

//	Now, issue the query
var ret = ADL.XAPIWrapper.getStatements(quizParams);
```

The first line creates the record with all of the parameters we'll send to the LRS in the query.  The next two lines set the Verb and Activity ID's we'll be searching for, respectively.  And the final line sends the query to the LRS.  That's it.  The returned array of statements will be assigned to the variable "ret."
