# Anatomy-of-the-Bot-Framework
_Anatomy of the Bot Framework. The purposes of this project is to describe the different pieces of the Bot Framework v4._

---
The Bot Framework v4 changed significantly from the previous version, v3.

Bot Framework v3 was great -- it was clear, easy to set up, and easy to use!
However, it was pretty opinionated and trying to do custom things sometimes felt like "swimming upstream" or "fighting the framework".

The Bot Framework v4 is different.  I think of the Bot Framework v4 as the Bot Framework "deconstructed" -- 
the team took what was once simple but blackboxed, and exposed a lot of the moving pieces -- giving you flexibility and a logical place to add custom code.

However, getting used to having all those pieces can be a lot at first.

The purposes of this project is to to describe the different pieces of the Bot Framework v4.

```
1) Channels
2) IBot
3) Dialogs
4) Middleware
5) Accessor
6) Startup.cs
```
### 1) Channels

The channel is where users can interact with your Bot.  You can add Bots to Slack, Facebook messenger, Microsoft Teams, and websites.

The typical interaction would be the users sending text messages, sending images, and selecting from various choice prompts buttons. The typical response from the Bot back to the user would be text, an html link of some sort, a stylized card, or a choice prompt.

The *type* of a message would be ActivityTypes.Message.

However, there are several different types of activities that the Bot can recognize including when a Bot joins the conversation and when a new user joins the conversation: https://github.com/Microsoft/BotBuilder/blob/master/specs/botframework-activity/botframework-activity.md

Most of your interactions will be messages back and forth between your Bot and the user; however, you can make your bot respond to events (as an example, your Bot might greet your user the first time the user joins the channel.)

### 2) IBot

When you hear the term Bot - it generally refers to the services that responds to users (including your code), the various backend services your Bot calls out to, and its expression in the channel.

However, in the anatomy of your Bot project app there will be a class where the heart of your of Bot will live; it will be a class that implmements the IBot interface.  This will typically be the main "control" point of your Bot.

Take a look at this simplified IBot class:
```
public class SimplifiedEchoBot : IBot
{
  public async Task OnTurnAsync(ITurnContext turnContext, CancellationToken cancellationToken = default(CancellationToken))
  {
    if (turnContext.Activity.Type == ActivityTypes.Message)
    {
      var responseMessage1 = $"You sent '{turnContext.Activity.Text}.'";
      await turnContext.SendActivityAsync(responseMessage1);
    }
  }
}
```
This SimplifiedEchoBot implements the IBot interface and has the important OnTurnAsync method which is what your Bot is going to do each time it receives an activity like a message from a user.  For each Activity received  (activity being a person or a bot joins the conversation or the person sends a message), a new instance of this class is created. Objects that are expensive to construct, or have a lifetime beyond the single turn, should be carefully managed.

For each user interaction, an instance of this class is created and the OnTurnAsync method is called.  This is a Transient lifetime service.  Transient lifetime services are created each time they're requested.  So the main "control point" of your Bot will be from this OnTurn method.

### 3) Dialogs

Now we have an IBot class which is the main "control point" of your Bot; how then do you code & structure the various conversations that your bot will have?

This is where "dialogs" come in. Dialogs are simply managed conversation flows in code. 

They are "managed" meaning that you define via code the back-and-forth interaction with the user with things like text prompts, clickable choices, or pre-defined series of steps (waterfall).

This requires you to pre-program the dialog. Does that sound too rigid? You can define multiple dialogs and appropriately trigger the right dialog.

Dialogs live in a stack.  The top dialog is the one being interacted with and so by changing what is on the stack or which ones is the top dialog is how you control the conversation. (ie. you can pop off all dialogs from the stack or you can replace the top dialog on the stack.)

Before you use any dialog, you have to let the Bot know which dialogs are available and put them in a set. 

In the constructor of your Bot, you'll have a DialogSet and you'll add the Dialogs to that DialogSet. You'll see the DialogSet being defined.  You'll also see a dialogContext being created from that dialog set which helps figure out which dialog is on top of the stack, and which dialogs are active etc etc.

```
public class DialogueBotWithAccessor : IBot
{
  private readonly DialogSet _dialogSet;
  private readonly DialogueBotConversationStateAccessor _accessors;

  public DialogueBotWithAccessor(DialogueBotConversationStateAccessor accessors)
  {
    _accessors = accessors ?? throw new ArgumentNullException(nameof(accessors));
    _dialogSet = new DialogSet(_accessors.ConversationDialogState);
    _dialogSet.Add(new TextPrompt("name"));
  }
  ...
```

### 4) Middleware

What is Middleware?  Think of it as a place in your project where you can add custom code before *and* after your bot processes a message. (Almost like global event handling.)  It's custom - so it can be anything you'd need but example functionality can include logging messages, listening for specific phrases, and running messages through APIs like sentiment using Azure's Text Analysis.

Timing-wise when does the Middleware get triggered?  <br />The below diagram shows you generally how turns function: <br/>

| Read Left to Right|-->|-->|-->|--> | 
| :-------------: | :-------------:| :-----:|:-------------:| :-----:|
| User sends message&nbsp; &nbsp;&nbsp;| Middleware 1 | Middleware 2 | Middleware 3  | OnTurnAsync() called |


| <--           | <--           | <--   |  <--  | Read Right To Left       |
| :-------------: |:---------------:|:-------:|:-------:|:--------------------------:|
| User receives message| Middleware 1 | Middleware 2 | Middleware 3 | OnTurnAsync() called |

### 5) Accessor

So we talked about your Bot class (the class that implements the IBot interface) being instantiated on each turn (or each time an activity is received.)  And we read that object that have a lifetime beyond the single turn, should be carefully managed.

So what we need is persistance that lives beyond the turn and we need a way to access that persisted data - ie. the Accessor. 

To make the "anatomy" of the problem clear -- suppose you were to create a Dialog without using persistance or accessors to access the persisted data.  In order to initiate the Dialog, you needed to create a DialogSet and DialogContext (without the appropriate persistance).  So each time the control flow happened: User input --> middleware --> Bot  --> Dialog the Bot was be re-instanced and we would lose any turn-to-turn information. (ie. Nothing would be persisted and would actually repeat the first step over and over.)

To fix this, we need to create persistance to the conversation and accessors to access the persisted data.

Such an example exists here that go from a Dialog Without Accessor to one with an Accessor:
Dialog Without Accessor:https://github.com/andrewchungxam/Mechanics-of-the-Bot-Framework-v4/tree/master/DialogWithoutAccessorBotV4 <br />
Dialog With Accessor: https://github.com/andrewchungxam/Mechanics-of-the-Bot-Framework-v4/tree/master/DialogWithAccessorBotV4 <br />

Here are the important stages of the transformation.  There are three example files that can be helpful that you can reference with the below explainations:<br />

Accessors class:
https://github.com/andrewchungxam/Mechanics-of-the-Bot-Framework-v4/blob/master/DialogWithAccessorBotV4/BotAccessor/DialogueBotConversationStateAccessor.cs <br />

IBot class with appropriate Accessors:
https://github.com/andrewchungxam/Mechanics-of-the-Bot-Framework-v4/blob/master/DialogWithAccessorBotV4/Bots/DialogsWithAccessor.cs <br />

Startup.cs where you can initialize options including storage, accessors, and middleware:
https://github.com/andrewchungxam/Mechanics-of-the-Bot-Framework-v4/blob/master/DialogWithAccessorBotV4/Startup.cs <br />



CREATING PERSISTANCE + ACCESSORS:<br/>
Keeping track of the dialog state requires a "chain" of pieces that ultimately starts with an object of type IStorage:
> Creating a DialogSet requires a Conversational Dialog State which keeps keeps track of the order in the stack of dialogs.

> Creating a DialogState requires a Conversation State which persists anything at the conversation level. (From the Conversation State, we created a property of Type DialogState.) 

> Creating a Conversation State requires an object of Type IStorage. 

In the Bot projects -> Startup.cs file is where we'll create persistance and give the bot access to the the accessor).<br/><br />
We're going to add the following pieces:<br/>
i. new MemoryStorage - this is how persistance is managed at the application level.  <br/>
ii.  new ConversationState(memoryStorage object) added to the options<br/>
iii. Add Singleton --> ConversationState from previous step is referenced and then added as a property to the accessor we need to create.<br/>
iv. The accessor is called DialogBotConversationStateAccessor and if you look at the full definition of the class DialogBotConversationStateAccessor.cs it has a property called ConversationDialogState.<br/>
```
public IStatePropertyAccessor<DialogState> ConversationDialogState { get; set; }
```
So what is the <DialogState> referring to?  This is defined in the BotFramework as a stack of dialogs.

This is the defintion of the DialogState.  Notice how it has a list of Dialog Instances
```
public class DialogState
{    
  public DialogState();
  public DialogState(List<DialogInstance> stack);
  public List<DialogInstance> DialogStack { get; }
}
```
Now via dependency injection, this accessor from Startup.cs is handed off to the Bot class which is recreated each turn
 -- this is how persistence is possible across turns.<br/><br/>

USING PERSISTANCE + ACCESSORS:<br/>
Now that we've defined and created them, let's look at how they are used -- go to file Bots > DialogueBotWithAccessor.cs

i. The constructor of the Bot takes as a parameter an object of type DialogueBotConversationStateAccessor.  
```  
private readonly DialogSet _dialogSet;
private readonly DialogueBotConversationStateAccessor _accessors;

public DialogueBotWithAccessor(DialogueBotConversationStateAccessor accessors)
{
  _accessors = accessors ?? throw new ArgumentNullException(nameof(accessors));
  _dialogSet = new DialogSet(_accessors.ConversationDialogState);
  _dialogSet.Add(new TextPrompt("name"));
}
```
This is the accessor we were setting up in Startup.cs.  The dependency injection will make sure the bot gets this each time it is instanced on each turn.

Notice there is a dialogSet that is instantiated and **notice it is taking as a parameter a property from the accessor**.  As an exercise, please right click on each of the classes/properties in the constructor to familiarize yourself with the specific definitions but the main point is that it needs a persistent list of Dialog instances.

Later within the OnTurn method --> from that dialogSet, we create a dialogContext
```
var dialogContext = await _dialogSet.CreateContextAsync(turnContext, cancellationToken);
var dialogTurnResult = await dialogContext.ContinueDialogAsync(cancellationToken);
```
And from the dialogContext --> we run ContinueDialog (ie. go to the next step if it can)

If that dialogStatus is Empty --> we know there wasn't a dialog called to begin with so we need to begin a new one.
We'll define one and call one directly off that dialogContext:
```
dialogContext.PromptAsync(
  "name", new PromptOptions { Prompt = MessageFactory.Text("STEP 4: This is the TextPrompt Dialog ::: PLEASE ENTER YOUR NAME.") },cancellationToken);

// We had a dialog run (it was the prompt) . Now it is Complete.
else if (dialogTurnResult.Status == DialogTurnStatus.Complete)
{
  // Check for a result.
  if (dialogTurnResult.Result != null)
  {
  // Finish by sending a message to the user. Next time ContinueAsync is called it will return DialogTurnStatus.Empty.
  await turnContext.SendActivityAsync(MessageFactory.Text($"THANK YOU, I HAVE YOUR NAME AS: '{dialogTurnResult.Result}'."));
  }
}
```
The final piece to the puzzle is this last call to save changes to the Conversation State which handles persistence of a conversation state object using the conversation ID
```
// Save changes if any into the conversation state.
await _accessors.ConversationState.SaveChangesAsync(turnContext, false, cancellationToken);
```


For example, the "MemoryStorage" object and associated "IStatePropertyAccessor{T}" object that are created with a singleton lifetime.  "https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-2.1"/


### 6) Startup.cs




```
REFERENCE PROJECTS:
1) Reference01-Bot Builder Echo Bot V4
2) Reference02-Bot Builder Basic Bot V4

PROJECTS:
1) 01 SimplifiedEchoBotV4
2) 02 SimplifiedMiddlewareEchoBotV4
3) 03 WelcomeMessageWithoutAccessorBotV4
4) 04 DialogWithoutAccessorBotV4
5) 05 DialogWithAccessorBotV4
6) 06 WelcomeMessageWithAccessorBotV4
7) 07 SimplifiedWaterfallDialogBotV4
8) 08 WaterfallDialogWithHardcodedCasesBotV4
9) 09 DuelingDialogsBotV4
10) 10 DuelingDialogsWithAccessorBotV4
11) 11 MultiDialogsWithAccessorBotV4

REFERENCE PROJECTS:
3) Reference03-ReferencesToImportantSamples
```

---
PREREQUISITES:
Before we run through the above projects -- make sure you've done the installation, and downloaded the templates and emulator:
https://docs.microsoft.com/en-us/azure/bot-service/dotnet/bot-builder-dotnet-sdk-quickstart?view=azure-bot-service-4.0

Also for the later AI projects (Reference03-ReferencesToImportantSamples) you'll want the Bot Builder CLI tools:
https://github.com/Microsoft/botbuilder-tools

Look for the word "Pre-requisite" - it will link to these sites:
https://nodejs.org/en/
https://www.microsoft.com/net/download

Once you've got the above prerequesites you can type in your command prompt: 
```
npm install -g chatdown msbot ludown luis-apis qnamaker botdispatch luisgen
```
(If you are having issues try opening your Powershell in Administrator Mode): <br />

---
### REFERENCE PROJECTS:
### 1) Reference01-Bot Builder Echo Bot V4 & 2) Reference02-Bot Builder Basic Bot V4

Take a quick glance at these two projects.  They are the template projects that get created from the Microsoft VSIX Bot templates.  (You'll have seen them in the installation instructions here: https://docs.microsoft.com/en-us/azure/bot-service/dotnet/bot-builder-dotnet-sdk-quickstart?view=azure-bot-service-4.0)

You'll notice there are a lot of pieces of the project --> don't worry too much about the details, we'll cover the pieces progressively through the various projects, just take about 10 minutes to look through the projects to get a general idea of the structure.

Start here:  <br /> Program.cs --> Startup.cs --> Files or folders that say Accessors --> Files or folders that say Bots --> Files or folders that say Dialogs.

Also, take quick look at BotConfiguration.Bot which has details of the project in Visual Studio (or open with Notepad).   If you double click on the file, Windows will open the Bot Framework Emulator.

### PROJECTS:
### 1) 01 SimplifiedEchoBotV4

* This bot takes the Echo Bot template and removes all the extraneous parts and simplifies it to the most basic part - the Bot.
For this project - we're only going to look at the SimplifiedEchoBot.cs -- this is where your Bot "lives".  

This SimplifiedEchoBot subclasses IBot and has the important OnTurnAsync method which is what your Bot is going to do each time it receives an activity like a message from a user.  OnTurn async is a transient object that gets created each time an activity is received (activity being a person or a bot joins the convo or the person sends a message.)

Take a quick look look at Startup.cs --> the only thing to notice for now is this line:
```
services.AddBot<SimplifiedEchoBot>(options =>    
```
which is how you add the SimplifiedEchoBot to your project.

Exercise:
* Go ahead and run the project and open it in your emulator.  Make a small modification of the return message and re-run the sample.

### 2) 02 SimplifiedMiddlewareEchoBotV4



### 3) 03 WelcomeMessageWithoutAccessorBotV4

The projects are meant to be progressive so you can see how each piece builds on the previous ones.
Two old things are happening in this Bot and two new things are happening in this Bot.

Old things:
1) Middleware is being triggered and sending messages to the user.  
2) The Bot is echoing the user.

New things:<br />
1) In the Bot code -- there is a bool DidBotWelcomeUser flag; this is toggled true/false in the bot code.  If false --> greet the user, then toggle to true.<br />
2) Also - the bot is checking to see if there is an ActivityType.ConversationUpdate.  This means it will check to see if somebody knew joined the channel.<br />

Exercises:
* Notice when the bot opens up, the statement: 
```
else if (turnContext.Activity.Type == ActivityTypes.ConversationUpdate)
```
is returning true more than once  - can you figure out why?
* Notice the welcoming message + bool flag system is not functioning correctly - can you figure out why?  We'll see examples of how to do this correctly in subsequent projects.

Now the OnTurn 

### 5) 05 DialogWithAccessorBotV4

In the previous project, there was no application level persistance to the DialogSet/DialogContext -- so each time the control flow happened, User input --> middleware --> Bot  --> Dialog, the Bot was be re-instanced and we lost any turn-to-turn information. (ie. Nothing was persisted.) The net result was that the bot would repeat the first step in the dialog over and over.

To fix this, we need to create persistance to the conversation and accessors to access the persisted data.



To make this clear -- suppose you were to create a Dialog without using persistance or accessors to access the persisted data.  In order to initiate the Dialog, you needed to create a DialogSet and DialogContext (without the appropriate persistance).  So each time the control flow happened: User input --> middleware --> Bot  --> Dialog the Bot was be re-instanced and we lost any turn-to-turn information. (ie. Nothing was persisted.) The net result was that the bot would repeat the first step in the dialog over and over.



CREATING PERSISTANCE + ACCESSORS:<br/>
Keeping track of the dialog state requires a "chain" of pieces that ultimately starts with an object of type IStorage:
> Creating a DialogSet requires a Conversational Dialog State which keeps keeps track of the order in the stack of dialogs.

> Creating a DialogState requires a Conversation State which persists anything at the conversation level. (From the Conversation State, we created a property of Type DialogState.) 

> Creating a Conversation State requires an object of Type IStorage. 


So let's start adding the necessary pieces.  Look at Startup.cs (this is where we'll create persistance and give the bot access to the the accessor).<br/><br />
We're going to add the following pieces:<br/>
i. new MemoryStorage - this is how persistance is managed at the application level.  <br/>
ii.  new ConversationState(memoryStorage object) added to the options<br/>
iii. Add Singleton --> conversation state from previous step is referenced and then added as a property to the accessor we created.<br/>
iv. The accessor is called DialogBotConversationStateAccessor and if you look at the full definition of the class DialogBotConversationStateAccessor.cs it has a property called Conversation Dialog State.<br/>
```
public IStatePropertyAccessor<DialogState> ConversationDialogState { get; set; }
```
So what is the DialogState referring to?  This is defined in the BotFramework as a stack of dialogs.

This is the defintion of the DialogState.  Notice how it has a list of Dialog Instances
```
public class DialogState
{    
  public DialogState();
  public DialogState(List<DialogInstance> stack);
  public List<DialogInstance> DialogStack { get; }
}
```
Now via dependency injection, this accessor from Startup.cs is handed off to the Bot class which is recreated each turn
 -- this is how persistence is possible across turns.<br/><br/>

USING PERSISTANCE + ACCESSORS:<br/>
Now that we've defined and created them, let's look at how they are used -- go to file Bots > DialogueBotWithAccessor.cs

i. The constructor of the Bot takes as a parameter an object of type DialogueBotConversationStateAccessor.  
```  
private readonly DialogSet _dialogSet;
private readonly DialogueBotConversationStateAccessor _accessors;

public DialogueBotWithAccessor(DialogueBotConversationStateAccessor accessors)
{
  _accessors = accessors ?? throw new ArgumentNullException(nameof(accessors));
  _dialogSet = new DialogSet(_accessors.ConversationDialogState);
  _dialogSet.Add(new TextPrompt("name"));
}
```
This is the accessor we were setting up in Startup.cs.  The dependency injection will make sure the bot gets this each time it is instanced on each turn.

Notice there is a dialogSet that is instantiated and **notice it is taking as a parameter a property from the accessor**.  As an exercise, please right click on each of the classes/properties in the constructor to familiarize yourself with the specific definitions but the main point is that it needs a persistent list of Dialog instances.

Later within the OnTurn method --> from that dialogSet, we create a dialogContext
```
var dialogContext = await _dialogSet.CreateContextAsync(turnContext, cancellationToken);
var dialogTurnResult = await dialogContext.ContinueDialogAsync(cancellationToken);
```
And from the dialogContext --> we run ContinueDialog (ie. go to the next step if it can)

If that dialogStatus is Empty --> we know there wasn't a dialog called to begin with so we need to begin a new one.
We'll define one and call one directly off that dialogContext:
```
dialogContext.PromptAsync(
  "name", new PromptOptions { Prompt = MessageFactory.Text("STEP 4: This is the TextPrompt Dialog ::: PLEASE ENTER YOUR NAME.") },cancellationToken);

// We had a dialog run (it was the prompt) . Now it is Complete.
else if (dialogTurnResult.Status == DialogTurnStatus.Complete)
{
  // Check for a result.
  if (dialogTurnResult.Result != null)
  {
  // Finish by sending a message to the user. Next time ContinueAsync is called it will return DialogTurnStatus.Empty.
  await turnContext.SendActivityAsync(MessageFactory.Text($"THANK YOU, I HAVE YOUR NAME AS: '{dialogTurnResult.Result}'."));
  }
}
```
The final piece to the puzzle is this last call to save changes to the Conversation State which handles persistence of a conversation state object using the conversation ID
```
// Save changes if any into the conversation state.
await _accessors.ConversationState.SaveChangesAsync(turnContext, false, cancellationToken);
```
### 6) 06 WelcomeMessageWithAccessorBotV4

At this point - you've gotten a feel of how the Middleware works.  
The messages from the Middleware have now been commented out: 
```
//await turnContext.SendActivityAsync($"STEP 1: MIDDLEWARE - BEFORE ");
```
If at any point through the next projects you want to see those messages and how they interact with the Bot - you can uncomment them.

In this project, you can see specifically how you can send two different types of welcome messages.  
You've tried this before; we'll do it successfully this time.

You'll send one message when the user joins the channel and one after the very first time the user send a message to the bot.
These are two different events and you can use them accordingly in the bot as needed.

WHEN THE USER JOINS THE CHANNEL (YOU'VE SEEN THIS ALREADY)<br/>
In WelcomeMessageWithAccessorBot.cs, there is a section underneath:<br/>
```
else if (turnContext.Activity.Type == ActivityTypes.ConversationUpdate)
```
which is devoted to sending a welcome message.  

You've seen this before -- this will send a message to the user as soon as the user joins the conversation.   It also sends a message when the bot joins the conversation -- to turn that off, uncomment this:
```
 //if (member.Id != turnContext.Activity.Recipient.Id) //IE. DON'T SEND THIS TO THE BOT
```
AFTER THE USER TYPES FIRST MESSAGE (THIS IS NEW)
```
var didBotWelcomeUser = await _dialogBotConversationStateAndUserStateAccessor.WelcomeUserState.GetAsync(turnContext, () => new WelcomeUserState());
```
As opposed to a bool flag that lives in the bot (which we tried and showed that it didn't work) - here the check is against the bool contained in property "WelcomeUserState" in the accessor.

Because it is part of the accessor, it is persistent.

Here is where the accessor is passed into the bot: 
```
public WelcomeMessageWithAccessorBot(DialogBotConversationStateAndUserStateAccessor accessor)
{
  _dialogBotConversationStateAndUserStateAccessor = accessor ?? throw new ArgumentNullException(nameof(accessor));
  _dialogSet = new DialogSet(_dialogBotConversationStateAndUserStateAccessor.ConversationDialogState);
  _dialogSet.Add(new TextPrompt("name"));
}
```
And it is setup to be passed in each time to the Bot via dependency injection -- you set it up here in the Startup.cs file: 
```
services.AddSingleton(sp =>
{
  ....
  // The dialogs will need a state store accessor. Creating it here once (on-demand) allows the dependency injection
  // to hand it to our IBot class that is create per-request.
  var accessors = new DialogBotConversationStateAndUserStateAccessor(conversationState, userState)
  {
    ConversationDialogState = conversationState.CreateProperty<DialogState>("DialogState"),
    WelcomeUserState = userState.CreateProperty<WelcomeUserState (DialogBotConversationStateAndUserStateAccessor.WelcomeUserName),
  };
    return accessors;
 });
```
Exercise:
* Go back to the previous project and file: <br /> WelcomeMessageWithoutAccessorBot > Bots > WelcomeMessageWithoutAccessorBot.cs - 
take a look at how we the welcome messages with the bool flag.  

### 7) 07 SimplifiedWaterfallDialogBotV4

A waterfall dialog is the most common and simplest way to piece together a cohesive conversation.

A waterfall dialog is defined by a list of steps which you define in an array 
(Go to file: SimplifiedWaterfallDialogBot.cs and take a look at the declaration var waterfallSteps = new WaterfallStep[] ... ).

Then within the dialogSet that declares which dialogs you'll be using in your bot, 
you add the WaterfallDialog named with a string of your choice and the waterfall steps array.  

In addition, you add any additional dialogs that you want in your conversation.  
```
_dialogBotConversationStateAndUserStateAccessor = accessor ?? throw new ArgumentNullException(nameof(accessor));
_dialogSet = new DialogSet(_dialogBotConversationStateAndUserStateAccessor.ConversationDialogState);

// This array defines how the Waterfall will execute.
var waterfallSteps = new WaterfallStep[]
{
  FirstStepAsync,
  NameStepAsync,
  NameConfirmStepAsync,
  AgeStepAsync,
  ConfirmStepAsync,
  SummaryStepAsync,
};

_dialogSet.Add(new WaterfallDialog("details", waterfallSteps));
_dialogSet.Add(new TextPrompt("name"));
_dialogSet.Add(new NumberPrompt<int>("age"));
_dialogSet.Add(new ConfirmPrompt("confirm"));
```
Exercises: 
* Look through the steps, look at how dialogs are interspersed throughout to create prompts and to gather user input. 
* Look at how input is extracted from one step of the waterfall to the next. 
* Comment out the following line in code (in SimplifiedWaterfallDialogBot.cs near the bottom of the OnTurnAsync method).  What issue arises? What, if anything, do you notice?
```
await _dialogBotConversationStateAndUserStateAccessor.UserState.SaveChangesAsync(turnContext, false, cancellationToken)
."), cancellationToken);  
``` 

### 8) 08 WaterfallDialogWithHardcodedCasesBotV4

WaterfallDialogWithHardcodedCasesBotV4 is similar to the project SimplifiedWaterfallDialogBotV4 with the exception that within the bot, it is listening to specific hardcoded phrases.

Look within the Bot for the section:
```
var text = turnContext.Activity.Text.ToLowerInvariant();
switch (text)
{
  ...
}
```
Exercises:
* What use cases can you think of that would require the checking of specific phrases?

### 9) 09 DuelingDialogsBotV4

We've talked about Bots, Middleware, Accessors, a single Dialog, and a single Waterfall Dialog.

In this sample, we're going to setup multiple dialogs and see how they interact with each other.

There are three dialogs: RootWaterfallDialog, ColorWaterfallDialog, and the FoodWaterfallDialog.

Exercises look through the code in DuelingDialogBot.cs
* Notice the large number of dialogs that are used -- remember adding them here to the dialogSet allows them to be kept track of by your application.
* Look at how the if statement is used to either BeginDialogAsync if there is no active dialog or ContinueDialogAsync to continue a dialog that has already begun.

Look through the code of RootWaterfallDialog, ColorWaterfallDialog, and FoodWaterfallDialog:
* Notice how they are seperate yet still are launched from each other.  (Given that they are now starting to seperate ... how can we pass values from one Dialog to the next?) 
* Notice how the two dialogs ColorWaterfallDialog and FoodWaterfallDialog end themselves and control is passed back to the RootWaterfallDialog.
* Notice how Rootwaterfall loops itself and is looped with delay.  How is this achieved? What does the delay add?  Play around with different delays and see how it affects the user experience.
* Middleware has now been commented out - but at the end of this exercise uncomment it to see how the Middleware and Dialogs work together.  (Note: At the prompt of "What would you like to talk about?" type out the choice - as an example: Favorite Food).

### 10) 10 DuelingDialogsUsingAccessorBotV4

We're dealing with multiple dialogs.  Data needs to pass back and forth between them; how do we do it?  

Try this: when choosing between Food + Color, try picking food first, then pick color.
Notice how the food value gets saved and recalled from state.

We'll need to use our Accessors - but before we had only one class that needed to access data (from our simplified Bot).
Now, the bot is growing in complexity; we'll need to set things up slightly differently so we can access them 
from various classes.

Declaring Accessors in Bot: <br/>
We'll create a public property instead of the private field we had before:<br/>
OLD:
```
//private readonly DialogBotConversationStateAndUserStateAccessor _dialogBotConversationStateAndUserStateAccessor;
```
NEW:
```
public DialogBotConversationStateAndUserStateAccessor DialogBotConversationStateAndUserStateAccessor { get; set; } 
//notice how it gets assigned in the constructor.
```
Using Accessor in Bot:
For example, in ColorWaterfallDialog.cs
```
private async Task<DialogTurnResult> NameConfirmStepAsync(WaterfallStepContext stepContext, CancellationToken cancellationToken)
{
*OLD: In the previous project - we simply pulled values from the step context without saving.*
// WITHOUT SAVING STATE WITH ACCESSOR TO 'THEUSERSTATE'
// We can send messages to the user at any point in the WaterfallStep.
// await stepContext.Context.SendActivityAsync(MessageFactory.Text($"COLOR WATERFALL STEP 3: I like the color {stepContext.Result} too!"), cancellationToken);
// END-WITHOUT SAVING STATE WITH ACCESSOR TO 'THEUSERSTATE'

*NEW: In the current project - we still pull values from the context -- but also access the DialogBotConversationStateAndUserStateAccessor.*

// WITH SAVING STATE WITH ACCESSOR TO 'THEUSERPROFILE'
var botState = await (stepContext.Context.TurnState["DialogBotConversationStateAndUserStateAccessor"] as DialogBotConversationStateAndUserStateAccessor).TheUserProfile.GetAsync(stepContext.Context);
botState.Color = stepContext.Result.ToString();
await stepContext.Context.SendActivityAsync(MessageFactory.Text($"FOOD WATERFALL STEP 3: I like {botState.Color} {botState.Food} as well! "), cancellationToken);
// END-WITH SAVING STATE WITH ACCESSOR TO 'THEUSERSTATE'

return await stepContext.EndDialogAsync(null, cancellationToken);
}
```
Exercise:
* In the MultiDialogWithAccessorBot.cs file try commenting out this:
```
await _dialogBotConversationStateAndUserStateAccessor.UserState.SaveChangesAsync(turnContext, false, cancellationToken);
```
What do you notice?

### 11) MultiDialogsWithAccessorBotV4

Now you've seen how multiple dialogs can interact with each other.

This time, we'll add a twist.  We'll do the same project but this time, within the middleware, we'll check for a specific key phrase.

If the user types that key phrase, then we'll want to see the bot kickoff a different dialog.
(regardless of what's happening in the bot a the time!)

Take a look at the SimplifiedEchoBotMiddleware3.cs
```
if (turnContext.Activity.Type == ActivityTypes.Message && turnContext.Activity.Text == "name")
{
  var didTypeNameString = "name";
  
  // Update user state flag to reflect bot was given a specific prompt
  turnContext.TurnState.Add("didTypeName", didTypeNameString);
  await next(cancellationToken);
} 
...
```
In the TurnState, we'll add as a dictionary key pair "didTypeName" and didBotWelcomeUser which equals "name".
We're going to access this in the Bot.

In the bot MultiDialogWithAccessorBot.cs, we're checking the TurnState dictionary to see if it contains a key "didTypeName"

Under the OnTurnAsync method:
```
....
  //POP OFF ANY DIALOG FROM THE STACK IF THE "FLAG" IS SWITCHED 
  string didTypeNamestring = "";
  if (turnContext.TurnState.ContainsKey("didTypeName"))
  {
    didTypeNamestring = turnContext.TurnState["didTypeName"] as string;
  }

  if (didTypeNamestring == "name")
  {
    //OPTION 1:
    await dialogContext.CancelAllDialogsAsync();
  }
```
Then you'll see later the following logic:<br />
If the dialogs are popped off the stack and the dictionary key/value pair exists "didTypeName"/"name" --> Start the NameWaterfallDialog.<br />
If the dialogs are popped off the stack or didn't exist in the first place and the dictionary key/value pair is not there --> Start the RootWaterfallDialog else continue whatever dialog is happening. <br />

Exercise:
* Notice how the dialogs are canceled (dialogContext.CancelAllDialogsAsync).  There are two other options commented out below it.  

### REFERENCE PROJECTS:
### 3) Reference03-ReferencesToImportantSamples

If you look in the above project - you'll see one file that references three projects from the official Microsoft samples.
You now understand the mechanics now of how the dialogs work, these new referenced samples will show to incorporate AI into 
your projects.

I've added instructions that will hopefully make it as easy as possible to go through the samples; with the aim of showing how you can take these official samples and make them your own (ie adding your own cognitive services).

```
/*
REFERENCES TO IMPORTANT SAMPLES
In the above projects, we've gone through the Mechanics of the Bot Framework

The official samples offer various specific use cases and important integrations.

Here are 4 important projects of note:

A) Integration with QnA MAker
https://github.com/Microsoft/BotBuilder-Samples/tree/master/samples/csharp_dotnetcore/11.qnamaker

B) Integration with LUIS for Natural Language Processing
https://github.com/Microsoft/BotBuilder-Samples/tree/master/samples/csharp_dotnetcore/12.nlp-with-luis<br/>

C) Integration with Dispatch
https://github.com/Microsoft/BotBuilder-Samples/tree/master/samples/csharp_dotnetcore/14.nlp-with-dispatch<br/>
Enables integration of Multiple AI Services (e.g. like multiple QnA and LUIS projects)*

D) Cards UI Project
https://github.com/Microsoft/BotBuilder-Samples/tree/master/samples/csharp_dotnetcore/06.using-cards<br/>
As you build out your UI and presentation for your bot, this project will show you various ways you can present your info to your customer.

The full repo of official samples can be found here:
https://github.com/Microsoft/BotBuilder-Samples/tree/master/samples/csharp_dotnetcore

*/
```
---
#### Reference Projects: 3A) Integration with QnA Maker
https://github.com/Microsoft/BotBuilder-Samples/tree/master/samples/csharp_dotnetcore/11.qnamaker

For this sample and the below AI samples (LUIS and Dispatch) - we need a couple things:

i) an AI Service <br/>
ii) a BotConfiguration.bot file to tie that service to our Bot<br/>
iii) Configure the Sample to connect to your AI project<br/>

i) an AI Service<br/>
First we need to set up the AI service - in this case, it will be the QnA Maker which uses AI to help match user queries to 
answers you've populated.  The population of these question/answer pairs is simple -->
you can upload a Word document, Excel document, CSV file, PDF, or even point it to a exisiting website 
with Question and Answer format (like a company's FAQ page).

Go here for instructions on how to set up an AI service:
https://docs.microsoft.com/en-us/azure/cognitive-services/qnamaker/how-to/set-up-qnamaker-service-azure

ii) a BotConfiguration.bot file to tie that service to our Bot<br/>
Second, we need a BotConfiguration.bot file which includes information on the service we are using 
plus configuration information for our Bot.

The BotConfiguration.bot file is not included in the official samples 
(though they will likely include empty templates in the future.)

If you tried building the projects, you probably already noticed.

You have two options - one, you can patch together your own BotConfiguration.bot file based on the example templates 
seen on the documentation pages - for QnA Maker you can see it here: 
https://docs.microsoft.com/en-us/azure/bot-service/bot-builder-howto-qna?view=azure-bot-service-4.0&tabs=cs

If you're doing it this way you'd be create a file called BotConfiguration.Bot and copy the template from the above.
(It should start like this - you can copy it directly from the documentation page with spaces for your specific entries.)
``` 
{
  "name": "QnABotSample",
  "services": [
    {
.... 
}
```
However, this is a bit of a "magic string" experience -- 
the recommended approach is to use the CLI which will make all the necessary files for you.  

The CLI is the more robust way to go; however, the downside -- it does require a fair bit of installation.  

Here is the main page of the CLI tools:
https://github.com/Microsoft/botbuilder-tools

Look for the word "Pre-requisite" - it will take your here:
https://nodejs.org/en/
https://www.microsoft.com/net/download

Once you've got the prerequesites you can type in your command prompt: 
npm install -g chatdown msbot ludown luis-apis qnamaker botdispatch luisgen
(if you are having issues try opening your Powershell in Administrator Mode): 

Here are the commands you should run 
(it will take you through various steps - I've put my answers down as reference, you can change if needed including encryption options for the keys.):
```
msbot init 
> BotConfiguration
> http://localhost:3978/api/messages
> no
> no
```
(You've just created a file call BotConfiguration.Bot --> if you double click it, it will open up your Bot Emulator.  Instead,
try right clicking it and open it up with Notepad or similar.) 
IF YOU'RE LOOKING AT IT WITH NOTEPAD, CLOSE IT AFTER YOU LOOK AT IT...OTHERWISE CLI WON'T BE ABLE TO MODIFY IT AND WILL FAIL SILENTLY.

Next run this command (again it will take you through various steps -- template numbers left there as an example but have been modified from original!)
```
qnamaker init
>5dd1234234567891ebcasdf139asdf79b // access/subscription key - get this from Azure portal in your QnA Maker project, go to Keys, select either of the Keys (I used Key 1)  
>1b4fasdf-asdf-asdf-asdf-asdf1234e3e7 //knowledgebase Id
//get this from QnA Maker portal (qnamaker.ai) 
	> Select My Knowledge bases 
	> Click the approrpriate knowledge base 
	> Click Settings 
	> Scroll down to Deployment details 	
	> Click Postman 
	> the first line will be POST /knowledgebases/1b4fasdf-asdf-asdf-asdf-asdf1234e3e7/generateAnswer
	> You'll want the string after knowledgebases/ and before /generateAnswer (ie. 1b4fasdf-asdf-asdf-asdf-asdf1234e3e7)
>yes 
```
You've just created a file called .qnamakerrcc, you don't need to touch it but the CLI will use it to connect your service to your BotConfiguration.bot file.
You can take a look by opening with Notepad.

Final command:
```
qnamaker get kb --kbId "1b4fasdf-asdf-asdf-asdf-asdf1234e3e7 " --msbot | msbot connect qna --stdin
```
^ again the string in quotes above is the knowledgeBase Id which we got in the previous step

Take another look at your BotConfiguration.bot file - you'll notice that the QnA service has been added.
The final step - *within* the text of the BotConfiguration.bot, under services, change value for "name" to "development".

!!! You'll see "name" more than once, so make sure you're changing the one that looks like this (otherwise, you'll see errors pop up in the Startup.cs file):
```
"services": [
        {
            "type": "endpoint",
   "name": "development",
```
Save this "name" change step until the very end. If you've been following the above steps in order, you should be great!
If you're looking at with Notepad, I recommend closing it after looking at it.

Having trouble?  In the qnamaker.ai portal:<br/> 
* Did you "Save and Train" your QnA Maker Bot? (reflects changes you made)
* Did you "Test" it? (make sure your utterances matches the intent you want)
* Did you "Publish" it? (makes it accessible)

iii) Configure the Sample to connect to your AI project

In QnABot.cs change this line ~#25:<br />
```
//public static readonly string QnAMakerKey = "QnABot";
```
Change that string to the name of your QnA project: 
(ie. it would match the name you used for your project here: https://www.qnamaker.ai/Home/MyServices)

Now you're done!  Run your QnA MAker and test it in your emulator!  
Type a question related to one of your QnA Maker Questions and Answers.

---
#### Reference Projects: 3B) Integration with LUIS for Natural Language Processing
https://github.com/Microsoft/BotBuilder-Samples/tree/master/samples/csharp_dotnetcore/12.nlp-with-luis

LUIS uses AI to match an utterance from a user and using AI connects it to an Intent that you define.

For example, you might create an intent "Cancel" and you might create sample utterances (you need at least 5 utterances) like "No!" "Stop!" "Not that!" "Cancel" "Restart" "Please stop".<br />
Then when a user types, "Please stop doing that" then the AI will match with some level of confidence that utterance to "Cancel".

Similar to the above steps you'll need:<br />
i) an AI Service <br />
ii) a BotConfiguration.bot file to tie that service to our Bot<br />
iii) Configure the Sample to connect to your AI project<br />


i) an AI Service 

Under the section Create a LUIS app in the LUIS portal: 	
https://docs.microsoft.com/en-us/azure/bot-service/bot-builder-howto-v4-luis?view=azure-bot-service-4.0&tabs=cs#create-a-luis-app-in-the-luis-portal

ii) a BotConfiguration.bot file to tie that service to our Bot

In the above steps, we've pointed out links to install the CLI - we'll go through the commands here:

Here are the commands you should run (it will take you through various steps - I've put my answers down as reference, you can change if needed including encryption options for the keys.):
```
msbot init 
> BotConfiguration
> http://localhost:3978/api/messages
> no
> no
```
```
luis init
> 8c5basdfasdfasdfasdfasdfasdf6f95 // Authoring Endpoint --> Luis.ai / My apps / Click appropriate app > Manage > Application Information > Keys and Endpoints
> westus // Authoring Endpoint --> Luis.ai / My apps / Click appropriate app > Manage > Application Information > Keys and Endpoints > Scroll down to Bottom > See Region field (also it should also be the subdomain of the Endpoint)
> ef9fasdf-asdf-asdf-asdf-asdfasdf6a83 // Application Id --> Luis.ai / My apps / Click appropriate app > Manage > Application Information > Application Id
> 0.1 version --> Luis.ai / My apps / Click appropriate app > Manage > Application Information > Versions
```
This will create a .luisrc file (you don't need to do anything to this but if you'd like to look, you can use Notepad)
```
luis get application --appId "ef9fasdf-asdf-asdf-asdf-asdfasdf6a83" --msbot | msbot connect luis --stdin
```
^ this appId is the same as you entered in the CLI prompts

iii) Configure the Sample to connect to your AI project

On line ~#27 in LuisBot.cs
Change this:    
```
//public static readonly string LuisKey = "LuisBot";
```

To the name of your Luis project -> Mine looks like this:
```
public static readonly string LuisKey = "BotLuisBotMA7-a7b7";
```
You can get the name here: 
Luis.ai / My apps / Click appropriate app > Manage > Application Information > Display Name

Now you're done!  Run your LUISBot and test it in your emulator!  
Type an utterance related to one of your intents and the Bot will return the best matching intent along with the Intent.

#### Reference Projects: 3C) Integration with Dispatch
*Enables integration of Multiple AI Services (e.g. like multiple QnA and LUIS projects):* <br /> https://github.com/Microsoft/BotBuilder-Samples/tree/master/samples/csharp_dotnetcore/14.nlp-with-dispatch


i) an AI Service <br />
ii) a BotConfiguration.bot file to tie that service to our Bot<br />
iii) Configure the Sample to connect to your AI project<br />

Dispatch is the Bot's way of determining what your user is trying to say vis-a-vis the various AI tools you've setup.  
For example, suppose you have both a LUIS application and a QnAMaker. <br />
Let's say you had set up your LUIS application to help identify customer support questions, and the QnA Maker to answer questions about taxes on purchases. <br />
If your user types "When do I pay my taxes?" you want to the bot to recognize that it is best routed to the QnA Maker.   <br />
This is where Dispatch comes in. <br />
Behind the scenes - Dispatch will actually create a new LUIS application -- the CLI will take care of most of it for you,
you simply need to type in a few values in the prompts.

You'll need the following for the Dispatch service:
i) an AI Service 
ii) a BotConfiguration.bot file to tie that service to our Bot
iii) Configure the Sample to connect to your AI project

i) an AI Service 
In order to run this project - you're going to need more than 1 service so I will assume you've done both the QnAMaker and LUIS bot in the previous projects.

ii) a BotConfiguration.bot file to tie that service to our Bot
You need to run similar commands to what you've done in the previous project -- the only exception is that you don't need to run the 
msbot init command twice (this is the step that creates the BotConfiguration.bot file and of course you only need to create it once.  Also I changed the "name" value under services to "development" (as you had done in previous samples) at the very end of 
my configuration -- including the dispatch init commands which are explained in the next section, so you can wait until 
all the CLI commands are run before you do any "by hand" modifications to your BotConfiguration.bot file. ).

So assuming you've run (these are the commands I've run in order):
```
qnamaker init
msbot init
qnamaker get kb --kbId "1b4fasdf-asdf-asdf-asdf-asdf1234e3e7 " --msbot | msbot connect qna --stdin
luis init
luis get application --appId "ef9fasdf-asdf-asdf-asdf-asdfasdf6a83" --msbot | msbot connect luis --stdin
```
^ This you've seen before and again if you're looking at the BotConfiguration.bot file -- make sure to close it so the CLI
can change it if necessary (CLI will fail silently if it cannot modify your file).

New commands:<br />
In this command - you'll point to the BotConfiguration.bot file
```
dispatch init -bot C:\Github\Microsoft-Bot-Framework-v4-SupportingDocs\OfficialSample14.nlp-with-dispatch\BotConfiguration.bot --hierarchical false
>>>BotConfigurationDispatch.dispatch created
```

Again point to the BotConfiguration.bot file
```
dispatch create -b C:\Github\Microsoft-Bot-Framework-v4-SupportingDocs\OfficialSample14.nlp-with-dispatch\BotConfig
tion.bot | msbot connect dispatch --stdin
```
If you did it right -> there will be a bunch Export / Creating / Updating commands prompting and finally you'll see this:
```
[msbot] {
  "type": "dispatch",
  "name": "BotConfigurationDispatch",
  "appId": "your-app-id-asdf",
  "authoringKey": "8c5basdfasdfasdfasdfasdfasdf6f95",
  "version": "Dispatch",
  "region": "westus",
  "serviceIds": [],
  "id": "250"
}
```
Now look at your BotConfiguration.bot file.

You'll need to make the following changes:
1) 

OLD:
```
....
"services": [       
{
  "type": "endpoint",  
  "name": "BotConfiguration",          
....
```
NEW:
```
....
"services": [  
{
  "type": "endpoint",           
  "name": "development",             
....
```

2) In your BotConfiguration.bot file - you'll want to make the following change.

In my BotConfiguration.bot file - I have the app, the QnA service, the LUIS service with id as "151", "245", "217" respectively.  
Simply add that to the serviceIds field in your dispatch service.

OLD:
```
...
{
  "type": "dispatch",
  "name": "BotConfigurationDispatch",
  "appId": "your-app-id-asdf",
  "authoringKey": "8c5basdfasdfasdfasdfasdfasdf6f95",
  "version": "Dispatch",
  "region": "westus",
  "serviceIds": [],
  "id": "250"
}
...
```
NEW: (You're looking at the values for "serviceIds" in this snippet:)

```
{
  "type": "dispatch",
  "name": "BotConfigurationDispatch",
  "appId": "your-app-id-asdf",
  "authoringKey": "8c5basdfasdfasdfasdfasdfasdf6f95",
  "version": "Dispatch",
  "region": "westus",
  "serviceIds": [ "151", "245", "217"],
  "id": "250"
}
```

iii) Configure the Sample to connect to your AI project

1) In appsettings.json - make sure your botFilePath is set to the BotConfiguration.bot file:
```
{
  "botFilePath": "BotConfiguration.bot",
  "botFileSecret": ""
}
```
In NlpDispatchBot.cs - Near the top of the file, you're going to need to add the name of your Luis app, your Dispatch app, and your QnA app -- you can look at your BotConfiguration.bot file to see their names.  (You know how to find this, we covered this when you did the qnamaker init and the luis init in the CLI.)

REPLACE THIS WITH THE NAME OF YOUR LUIS APPLICATION NAME
OLD:<br />
```
private const string HomeAutomationLuisKey = "Home Automation";
```   
NEW:
```
private const string HomeAutomationLuisKey = "BotLuisBotMA7-a7b7";
```
REPLACE THIS WITH THE NAME OF YOUR DISPATCH APPLICATION NAME
OLD:<br />
```
private const string DispatchKey = "nlp-with-dispatchDispatch";
```
        
NEW:<br />   
```
private const string DispatchKey = "BotConfigurationDispatch";
```
REPLACE THIS WITH THE NAME OF YOUR QnA APPLICATION NAME
OLD:<br />
```
private const string QnAMakerKey = "sample-qna";
```
NEW:<br />
```
private const string QnAMakerKey = "RoyaltyInfo2018";
```

If you go into your Luis.ai portal you'll notice a new Luis application has been built for you for the Dispatch application.

Go into it and look at the intents. (under Build > Intents) 

You should see the intents that you had in your Luis application (ones you've defined and also some default ones) and also the QnA one.  
I'm going to include only two intents in this exercise.  For me I've define one for video game support -- 
the intent name is "VideoGames" and I'll support the QnA service which appears for me as "q_RoyaltyInfo2018".

In the file - NlpDispatchBot.cs - around line ~144 under DispatchToTopIntentAsync, I've changed the following. 
Please make corresponding changes based on the names of your *INTENTS* as defined within the Luis application automatically 
created for you when you ran dispatch create in the CLI.  (ie. Look in the Luis.ai portal > Go to the new Dispatch project >
Go to Build > Intents and grab the names of two Intents you care about.)

OLD:<br />
```
const string homeAutomationDispatchKey = "l_Home_Automation";
const string weatherDispatchKey = "l_Weather";
const string noneDispatchKey = "None";
const string qnaDispatchKey = "q_sample-qna";
```
NEW:<br />
```
const string homeAutomationDispatchKey = "VideoGames";
//const string weatherDispatchKey = "l_Weather"; // I've commented this out as I'm only focused on the Luis intent "VideoGames" and the QnA intent "q_RoyaltyInfo2018" 
const string noneDispatchKey = "None";
const string qnaDispatchKey = "q_RoyaltyInfo2018";
```
Finally - in the same file: NlpDispatchBot.cs - simply comment out references to the weatherDispatchKey as we will only be
using two services.  So comment out these sets of lines:

approximately line 73:<br />
```
//if (!_services.LuisServices.ContainsKey(WeatherLuisKey))
//{
//    throw new System.ArgumentException($"Invalid configuration. Please check your '.bot' file for a Luis service named '{WeatherLuisKey}'.");
//}
```
and approximately line 164:
```
//case weatherDispatchKey:
//    await DispatchToLuisModelAsync(context, WeatherLuisKey);

//    // Here, you can add code for calling the hypothetical weather service,
//    // passing in any entity information that you need
//    break;
```
As reference, the official doc for the Dispatch project is here: https://docs.microsoft.com/en-us/azure/bot-service/bot-builder-tutorial-dispatch?view=azure-bot-service-4.0&tabs=csharp
(it does not take you through using your own Luis / Dispatch models which we've done above)

Now you are done!  Run your Dispatch Bot and test it!  
Type an utterance related to one of your intents and the Bot will return the best matching intent or if it's from the QnA Maker it will return an answer!

#### REFERENCE PROJECTS: 3D) Cards UI Project
*As you build out your UI and presentation for your bot, this project will show you various ways you can present your info to your customer:* <br />
https://github.com/Microsoft/BotBuilder-Samples/tree/master/samples/csharp_dotnetcore/06.using-cards

Exercise:
* Run the sample and take a look at each card type and the corresponding code.

The full repo of official samples can be found here:
https://github.com/Microsoft/BotBuilder-Samples/tree/master/samples/csharp_dotnetcore of special further note is the Authentication Bot.



















