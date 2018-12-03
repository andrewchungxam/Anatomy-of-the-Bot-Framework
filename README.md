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

Accessors class:<br />
https://github.com/andrewchungxam/Mechanics-of-the-Bot-Framework-v4/blob/master/DialogWithAccessorBotV4/BotAccessor/DialogueBotConversationStateAccessor.cs <br />

IBot class with appropriate Accessors:<br />
https://github.com/andrewchungxam/Mechanics-of-the-Bot-Framework-v4/blob/master/DialogWithAccessorBotV4/Bots/DialogsWithAccessor.cs <br />

Startup.cs where you can initialize options including storage, accessors, and middleware:<br />
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

### 6) Startup.cs

Here are snippets of a sample Startup.cs class - full class here: <br />
https://github.com/andrewchungxam/Mechanics-of-the-Bot-Framework-v4/blob/master/DialogWithAccessorBotV4/Startup.cs

```
...
public void ConfigureServices(IServiceCollection services)
{
  services.AddBot<DialogueBotWithAccessor>(options =>
  {
...
    options.CredentialProvider = new SimpleCredentialProvider(endpointService.AppId, endpointService.AppPassword);
    
    options.Middleware.Add(new SimplifiedEchoBotMiddleware1());
    options.Middleware.Add(new SimplifiedEchoBotMiddleware2());
    options.Middleware.Add(new SimplifiedEchoBotMiddleware3());

    // Memory Storage is for local bot debugging only. When the bot
    // is restarted, everything stored in memory will be gone.  This is where you'd want to use Cosmos or Blob Storage to have persistance beyond application's life.
    IStorage dataStore = new MemoryStorage();
                
    // Create and add conversation state.
    var conversationState = new ConversationState(dataStore);
    options.State.Add(conversationState);
    });

    // Create and register state accessors.
    // Accessors created here are passed into the IBot-derived class on every turn.
    services.AddSingleton(sp =>
    {    
      // We need to grab the conversationState we added on the options in the previous step.    
      var options = sp.GetRequiredService<IOptions<BotFrameworkOptions>>().Value;
      if (options == null)
      {
        throw new InvalidOperationException("BotFrameworkOptions must be configured prior to setting up the State Accessors");
      }

      var conversationState = options.State.OfType<ConversationState>().FirstOrDefault();
      if (conversationState == null)
      {
        throw new InvalidOperationException("ConversationState must be defined and added before adding conversation-scoped state accessors.");
      }

      // The dialogs will need a state store accessor. Creating it here once (on-demand) allows the dependency injection
      // to hand it to our IBot class that is create per-request.
      // Create the custom state accessor.
      // State accessors enable other components to read and write individual properties of state.
      var accessors = new DialogueBotConversationStateAccessor(conversationState)
      {
        ConversationDialogState = conversationState.CreateProperty<DialogState>("DialogState"),
      };
      return accessors;
      });
    }
```

To read more about how Dependency Injection works for the Bot:  "https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-2.1"/

