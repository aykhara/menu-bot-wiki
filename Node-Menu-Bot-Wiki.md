This Wiki explains concepts and patterns present in the Node menu-bot. To see the bot code, take a look at the [javascript_nodejs](https://github.com/ryanvolum/menu-bot/tree/master/javascript_nodejs) directory in this repo. 

## Guiding Conversations 
At a high level, this project demonstrates how a bot can guide a conversation using a menuing system. The end user is always presented with a discrete set of options so that there is never any ambiguity around what they can do. Naturally, when we think about bot development we immediately think of using Natural Language Processing (NLP) to understand user _intent_ and extract relevant _entities_. However, if we rely entirely on NLP, users are often left in the dark about what they can say on a given step. When our modality (in this case a web client) allows it, we should _show_ users what they can do with rich controls like buttons, cards and carousels. We can _then_ layer NLP over our conversation to short circuit conversations. 

We construct the guided conversation by first welcoming the user, and then using Waterfall Dialog and Prompt abstractions (from the botbuilder SDK) to scaffold out our conversation. Here's a quick peek at one of this bot's conversation flows: 

![DonateDialog GIF](https://github.com/ryanvolum/menu-bot/blob/master/wiki_assets/DonateDialog.gif)

## Welcoming the User
If our bot doesn't proactively welcome a user, then the user won't have any sense for what the bot is capable of doing. Take a look at in-depth [botbuilder document](https://docs.microsoft.com/en-us/azure/bot-service/bot-builder-send-welcome-message?view=azure-bot-service-4.0&tabs=csharp%2Ccsharpmulti%2Ccsharpwelcomeback) for more details. In this case we welcome our user by listening for any incoming `ConversationUpdate` activities in our `onTurn` function, validating that a new member was added, sending a message, and starting a dialog: 
```js
if (turnContext.activity.type === ActivityTypes.ConversationUpdate) {
    if (this.memberJoined(turnContext.activity)) {
        await turnContext.sendActivity(`Hey there! Welcome to the food bank bot. I'm here to help orchestrate 
                                        the delivery of excess food to local food banks!`);
        await dialogContext.beginDialog(MENU_DIALOG);

```

The member joined function is just a helper to abstract away some of the complexity of determining whether a member joined the conversation (as opposed to the bot): 
```js
memberJoined(activity) {
    return ((activity.membersAdded.length !== 0 && (activity.membersAdded[0].id !== activity.recipient.id)));
}
```

## Waterfall Dialogs and Prompts
This sample uses the botbuilder's waterfall dialog and prompt abstractions to model its conversation. At a high level, a waterfall dialog is an array of functions to run step-by-step on each conversation turn. Waterfall dialogs are great for expressing rigid conversations, where a specific series of steps should always occur. They're not so good for expressing conversations with lots of context switching. In this case our "food bank bot" should follow a fairly regimented flow to get users to the information they need, so waterfall dialogs are a great choice!

Before setting up our dialogs, we need to make sure to have a place for them to save their state. Specifically, dialogs need to know where they are in a conversation in order to kick off the right step. Let's take a step back to show how we plumb our state management - if you've already set this up, feel free to skip ahead to the [Creating a Waterfall](https://github.com/ryanvolum/menu-bot/wiki#creating-a-waterfall-dialog) section. 

### Setting up State 
We create our `ConversationState` instance in our index.js by first creating a `MemoryStorage` instance and passing it to the `ConversationState` constructor:
```js
const memoryStorage = new MemoryStorage();
const conversationState = new ConversationState(memoryStorage);
```
We could have passed the `MemoryStorage` constructor a storage provider (Azure Tables, Azure Cosmos, etc.), which would allow `ConversationState` to then get and set state against that external provider. For now we'll keep it empty, which will instead store everything in memory. 

Next, we initialize our bot with that `ConversationState` instance: 
```js
const myBot = new FoodBot(conversationState);
```

### Creating a Waterfall Dialog
Now that our bot has a place to save its state, we create a `dialogState` property and a `DialogSet`. We make both of these properties of our bot class to best organize our bot: 

```js
constructor(conversationState) {
     this.conversationState = conversationState;
     this.dialogState = this.conversationState.createProperty(DIALOG_STATE_PROPERTY);
     this.dialogs = new DialogSet(this.dialogState);
```
Now we can create our waterfall dialog and add it to the `DialogSet`. A waterfall dialog has a unique string name for persisting and reusing it. In this case we're using the `MENU_DIALOG` variable (declared at the top of our code) as our dialog's name: 

```js
        this.dialogs.add(new WaterfallDialog(MENU_DIALOG, [
            this.promptForMenu,
            this.handleMenuResult,
            this.resetDialog,
        ]));
```
As mentioned above, a `WaterfallDialog` is just an array of functions that will run in series. This waterfall dialog is composed of three functions, each called a `step`. We could declare these functions anonymously (inline and without a name), but chose to refer to them this way to better organize our code. Let's take a look at the first function: 

```js
    async promptForMenu(step) {
        return step.prompt(MENU_PROMPT, {
            choices: ["Donate Food", "Find a Food Bank", "Contact Food Bank"],
            prompt: "Do you have food to donate, do you need food, or are you contacting a food bank?",
            retryPrompt: "I'm sorry, that wasn't a valid response. Please select one of the options"
        });
    }
``` 
In this function, we prompt the user with three choices, "Donate Food", "Find a Food Bank" and "Contact Food Bank". Just as our dialog had a name, we need to name prompts as well. In this case, we're calling this choice prompt by the name `MENU_PROMPT`. We also register this choice prompt in our bot's constructor, so that we can reuse it throughout the bot:
```js
this.dialogs.add(new ChoicePrompt(MENU_PROMPT));
```
When we call our prompt, we defined three properties: `choices`, `prompt`, and `retryPrompt`. `choices` is the array of options, which will ultimately render as `suggestedActions`: buttons that only exist for one turn. `prompt` is the text we actually want to prompt the users with. `retryPrompt` is the text that the bot will use if a user replies with something other than the button text. 

In our next dialog step, we parse the user's answer: 
```js
    async handleMenuResult(step) {
        switch (step.result.value) {
            case "Donate Food":
                return step.beginDialog(DONATE_FOOD_DIALOG);
            case "Find a Food Bank":
                return step.beginDialog(FIND_FOOD_DIALOG);
            case "Contact Food Bank":
                return step.beginDialog(CONTACT_DIALOG);
        }
        return step.next();
    }
```
As you can see, the value of the user's response shows up in `step.result.value`. We use a switch case to determine which answer they gave and begin the next dialog as necessary. Note that we have no `default` handler in our switch case. This is because our dialog will only ever get to this step if user entered one of the valid inputs (or clicked a button). 

Depending on which button was clicked, we call `step.beginDialog` with the name of other dialogs that we've defined. We'll get into how we defined those other dialogs in [Creating Component Dialogs](https://github.com/ryanvolum/menu-bot/wiki#creating-component-dialogs), but for now suffice it to say that `beginDialog` pushes another dialog onto a dialog stack such that subsequent messages get sent to the new dialog until it finishes. Once that dialog finishes, we come back to the last step of our Main Menu dialog: 

```js
async resetDialog(step) {
    return step.replaceDialog(MENU_DIALOG);
}
```
This one-line function enables us to build a "message loop". Basically, when we've finished a conversation flow we get to this step, which takes us back to the beginning of the main menu. We accomplish this by replacing the current Main Menu dialog with itself, using the `step.replaceDialog` function. The bot will now start back at the first step of the Main Menu dialog, accomplishing our goal of never leaving a user in the dark about what they can do on a specific turn. 

### Orchestrating Our Dialog from the onTurn Function
Now that we've created and added our `WaterfallDialog` to our `DialogSet`, we need to let our bot know when to start and continue it. Most of this code will be fairly boilerplate for any bot that uses dialogs. As a quick refresher, the `onTurn` function is the function that gets called on every single turn. That means any time we get a message from our user, we run the `onTurn` function. `onTurn` receives a context object as a parameter, which bundles up the incoming activity and several conversational helpers for orchestrating the conversation. Let's take a look at this bot's `onTurn` function. 

We start by creating a `dialogContext`: 
```js
    async onTurn(turnContext) {
        const dialogContext = await this.dialogs.createContext(turnContext);
```
This dialog context contains helpers to assess the state of our dialogs. In this case, we'll use it to determine if there is an active dialog, and to continue it if there is. Note that we only do this when we receive messages from the user (Message Activities):  

```js
        if (turnContext.activity.type === ActivityTypes.Message) {
            if (dialogContext.activeDialog) {
                await dialogContext.continueDialog();
...
```

If there is no active dialog, we go ahead and start our Main Menu dialog: 

```js
            } else {
                await dialogContext.beginDialog(MENU_DIALOG);
            }
```

When we start or continue a dialog from our `onTurn`, the dialog will run the appropriate step(s) (waterfall functions), and then return. It's therefore important for us to save all state changes at the end of our `onTurn` function. Remember that we're saving our dialog's state through our `ConversationState` instance. If we don't actually call `conversationState.saveChanges`, they won't be persisted and we'll never move on to subsequent dialog steps:

```js
        await this.conversationState.saveChanges(turnContext);
```
The rest of the `onTurn` function contains the welcome code which we looked at in [Welcoming the User](https://github.com/ryanvolum/menu-bot/wiki/Home/_edit#welcoming-the-user). 

### Creating Component Dialogs
Now that we've taken a look at our top-level waterfall dialog, let's dive into the other dialogs in this bot. If you navigate through the project directory, you'll find a folder called `dialogs` with three files: `DonateFoodDialog.js`, `FindFoodDialog.js` and `ContactDialog.js`. We declare these dialogs outside of our `bot.js` for a few reasons. For one, building all of our conversation flow in one file would get unmanageable. It would be near impossible to work collaboratively with other developers in that same file. Separating dialogs also allows us to treat them as reusable modules - we could use them multiple times in the same bot, or even publish them to be used in other bots. In order to achieve this modular behavior, we rely on `ComponentDialogs`, which act as a module for a dialog or multiple dialogs. Let's take a look at the `FindFoodDialog`. 

Our dialog inherits from `ComponentDialog` and takes a dialogId: 
```js
class DonateFoodDialog extends ComponentDialog {
    constructor(dialogId) {
        super(dialogId);
```

It then sets `this.initialDialogId` to the name of our waterfall dialog:

```js
// ID of the child dialog that should be started anytime the component is started.
this.initialDialogId = dialogId;
```

In this case our waterfall dialog will have the same name as our Component Dialog, since it's the only dialog in our Component Dialog. If we choose not to explicitly set `this.initialDialogId`, it will automatically be set to the first dialog added to the ComponentDialog. Next, we add a `ChoicePrompt` that we will be using in our waterfall dialog, just as we did in our Main Menu dialog: 

```js
this.addDialog(new ChoicePrompt('choicePrompt'));
```

And we then add our waterfall dialog: 

```js
this.addDialog(new WaterfallDialog(dialogId, [
    async function (step) {
        return await step.prompt('choicePrompt', {
            choices: getValidPickupDays(),
            prompt: "What day would you like to pickup food?",
            retryPrompt: "That's not a valid day! Please choose a valid day."
        });
    },
    async function (step) {
        const day = step.result.value;
        let filteredFoodBanks = filterFoodBanksByPickup(day);
        let carousel = createFoodBankPickupCarousel(filteredFoodBanks)
        return step.context.sendActivity(carousel);
    }
]));
```

Note that instead of declaring the steps of the waterfall as members of the class, we just coded them inline as anonymous functions. We took this approach here since we know we'll only use these functions once, and because the code footprint is fairly small. 

As with our Main Menu dialog, this waterfall is an array of functions. The first prompts users for an array of days (determined by a helper method that process a JSON schedule). The second handles the response by creating a carousel of cards that display food banks open on the selected day. To create cards and carousels, we use the `CardFactory` in the `botbuilder` package. See `schedule-helpers` to see the full implementation, and check out the [Add Media to Messages](https://docs.microsoft.com/en-us/azure/bot-service/bot-builder-howto-add-media-attachments?view=azure-bot-service-4.0&tabs=csharp) Azure doc for more information about Hero Cards and carousels. 

### Persisting State throughout a Dialog
Sometimes a dialog needs to persist some information to be used on a later step. Take a look at the Contact conversation flow below:

![ContactDialog GIF](https://github.com/ryanvolum/menu-bot/blob/master/wiki_assets/ContactDialog.gif)

 **Note**: this flow isn't actually sending a food bank a message, so feel free to test the bot yourself!

You can see that we gather the user's email address, the message they want to send and the name of the food bank they want to send it to. We then use all that information to send a message to a food bank. But how are we persisting this information throughout the lifetime of the dialog? 

Let's take a look at the `ContactDialog` dialog. Like our other Component Dialogs, we're anonymously creating an array of functions that the bot will run through one at a time: 
```
this.addDialog(new WaterfallDialog(dialogId, [
...
]
``` 
These functions prompt users for the name of the Food Bank they want to contact (using `ChoicePrompt`), their email address (using `TextPrompt`) and the message they want to send (also using `TextPrompt). It also asks them to confirm that they want to send a message, which uses `ConfirmPrompt`. When we gather a piece from the user that we know we'll need later, we save it to the `step.values` dictionary: 
```
// Persist the email address for later waterfall steps to be able to access it
step.values.email = step.result;
```
Then on later turns we can access that property on the same `step.values` dictionary!
```
sendFoodBankMessage(step.result.foodBankName, step.result.message, step.result.email);
```