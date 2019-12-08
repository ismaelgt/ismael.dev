
# Building cross-platform apps for mobile, web and voice (part II)

*In this article and its part I we explore the technologies available to build a cross-platform experience which includes voice assistants. Assuming we want to build a small and simple to-do list app for Android, iOS, web and the Google Assistant, how can we provide the best experience for each platform while making sure our app can scale?*

![](https://ismael.dev/static/articles/voxe/devices-final.jpg)

To bring to life the tech stack defined in the previous article, we’ve created VOXE (VOice X-platform Experience). VOXE is a simple proof-of-concept to-do list app for Android, iOS, web and the Google Assistant. One single repository, which contains:

* A NativeScript (Angular) app that compiles to Android, iOS and web.
* A Dialogflow agent.
* A Cloud Function for the Google Action fulfillment.

The app makes use of Cloud Firestore to store the data.

![VOXE architecture](https://ismael.dev/static/articles/voxe/voxe-architecture.png)
*VOXE architecture*

The repository is publicly available [on GitHub](https://github.com/potatolondon/voxe).

## Firebase: Infrastructure and services

VOXE makes use of three Firebase services:

* [Hosting](https://firebase.google.com/docs/hosting): for the web app.
* [Authentication](https://firebase.google.com/docs/auth): we will use Google here for simplicity but Firebase supports many other authentication methods such as passwords, phone numbers and other popular federated identity providers like Facebook, Twitter, etc.
* [Cloud Firestore](https://firebase.google.com/docs/firestore).

The first two services are easily configurable via the [Firebase Console](https://console.firebase.google.com/). Let’s dig into our data model and security rules.

### Cloud Firestore data model

Cloud Firestore is a NoSQL, document-oriented database. Unlike a SQL database, there are no tables or rows. Instead, the data is stored in *documents*, which are organised into *collections*.

We’ll need, then, a collection of to-do documents (*todos*), each of which will define the following fields: `content` (string), `createdAt` timestamp (number) and `userId` (string).

![](https://ismael.dev/static/articles/voxe/firestore-definition.png)

### Security rules

Since we’re relying on Firebase client SDKs and not on a bespoke API, how do we make sure users only have access to to-do items created by themselves?

Every database request from a Cloud Firestore mobile/web client library is evaluated against a set of [security rules](https://firebase.google.com/docs/firestore/security/get-started) before reading or writing any data. These rules define the level of access for a specific path in the database using conditions based on authentication and data fields.

Let’s have a look at the security rules for our database:

```
service cloud.firestore {
  match /databases/{database}/documents {
    match /todos/{todo} {
      allow read, delete: if request.auth.uid == resource.data.userId;
      allow create: if request.auth.uid == request.resource.data.userId;
      allow update: if request.auth.uid == resource.data.userId && request.auth.uid == request.resource.data.userId;
    }
  }
}
```

* `resource.data` represents the document accessed by the query.
* `request.auth.uid` is the `id` of the authenticated user.
* `request.resource.data` represents a new or updated document.

So this set of security rules will make sure that:

* Users can only read and delete documents whose `userId` field matches their user `uid`.
* User can only create documents whose `userId` field matches their user `uid`.
* User can only update documents whose `userId` field matches their user `uid` and this field can’t be changed.

Our database is now ready to roll.

## NativeScript: A cross-platform native and web app

In our previous article we discussed some of the benefits of using [NativeScript](https://www.nativescript.org) for building cross-platform apps, focusing on its [code-sharing](https://docs.nativescript.org/angular/code-sharing/intro) capabilities using Angular.

A code-sharing project is one where we keep the code for the web and mobile apps in one place. Let’s have a look at some examples of how we have benefited from this architecture on VOXE.

At **component** level, these files define the app homepage:

* [`home.component.ts`](https://github.com/potatolondon/voxe/blob/master/src/app/home/home.component.ts): Angular component definition (web and native).
* [`home.component.html`](https://github.com/potatolondon/voxe/blob/master/src/app/home/home.component.html): web template file.
* [`home.component.tns.html`](https://github.com/potatolondon/voxe/blob/master/src/app/home/home.component.tns.html): mobile template file.

![](https://ismael.dev/static/articles/voxe/nativescript-code-sharing-2.png)

It is also possible to use the naming convention for platform-specific **styling**:

* [`home.component.scss`](https://github.com/potatolondon/voxe/blob/master/src/app/home/home.component.scss): web styles.
* [`home.component.android.scss`](https://github.com/potatolondon/voxe/blob/master/src/app/home/home.component.android.scss): Android styles.
* [`home.component.ios.scss`](https://github.com/potatolondon/voxe/blob/master/src/app/home/home.component.ios.scss): iOS styles.

We’ve been able to write an implementation for the Home component ([`home.component.ts`](https://github.com/potatolondon/voxe/blob/master/src/app/home/home.component.ts)) that works both for web and mobile but this might not always be possible. In fact, for the Authentication and Firestore **services**, this isn’t possible since we’re forced to use the Firebase native client SDK for each platform. For this to work, again, we use the naming convention to define two different services. For example, the Firestore service is defined by these files:

* [`firestore.service.ts`](https://github.com/potatolondon/voxe/blob/master/src/app/firestore/firestore.service.ts): web implementation using the [`angularfire`](https://github.com/angular/angularfire) library.
* [`firestore.service.tns.ts`](https://github.com/potatolondon/voxe/blob/master/src/app/firestore/firestore.service.tns.ts): mobile implementation using [`nativescript-plugin-firebase`](https://github.com/EddyVerbruggen/nativescript-plugin-firebase).
* [`firestore.service.interface.ts`](https://github.com/potatolondon/voxe/blob/master/src/app/firestore/firestore.service.interface.ts): interface used by both implementations of the service.

## Actions on Google: The voice experience

As previously discussed, some of our app features could be triggered via voice (or text) commands using the Google Assistant. Let’s assume we want to be able to read our to-do list, add and remove items. Here’s a diagram of the conversation flow:

![](https://ismael.dev/static/articles/voxe/conversation-diagram.png)

In order to achieve this integration we’ll need to follow these three steps:

1. Create an Action in the [Google Actions Console](https://console.actions.google.com/).
1. Create a [Dialogflow Agent](https://dialogflow.cloud.google.com).
1. Deploy a [Fulfillment](https://developers.google.com/assistant/actions/dialogflow#build_and_deploy_fulfillment).

### Dialogflow

[Dialogflow](https://cloud.google.com/dialogflow/) is a natural language understanding platform that allows us to design and build conversational user interfaces. A [Dialogflow agent](https://cloud.google.com/dialogflow/docs/agents-overview) is a virtual agent that handles these conversations with your end-users. Although the focus of this article is on the Google Assistant, it’s important to mention that Dialogflow can also be used to build text chatbots that can be integrated with different platforms.

To describe how to do something we use [intents](https://cloud.google.com/dialogflow/docs/intents-overview). We want our Action to respond to requests like:

* Read my list
* Add a new item to my list
* Remove an item from my list

We can create these intents in Dialogflow. Here’s the list of Intents in VOXE:

![List of intents in VOXE](https://ismael.dev/static/articles/voxe/dialogflow-intents-list.png)
*List of intents in VOXE*

For each of the intents we need to provide [training phrases](https://cloud.google.com/dialogflow/docs/intents-training-phrases). These are example phrases for what end-users might type or say, referred to as *end-user expressions*. For each intent, we create many training phrases. When an *end-user expression* resembles one of these phrases, Dialogflow matches the intent. “Please read my list”, “Remove an item from my list” or “Add an item to my list” are examples of training phrases for intents in VOXE.

![Training phrases for “Add item request” intent](https://ismael.dev/static/articles/voxe/dialogflow-training-phrases.png)
*Training phrases for “Add item request” intent*

Training phrases can contain [parameters](https://cloud.google.com/dialogflow/docs/intents-actions-parameters) of different entity types, from which we can extract values. For example, when the user mentions the item to be added to, or removed from the list, this list item is extracted as a parameter.

![“item” parameter for “Add item request” intent](https://ismael.dev/static/articles/voxe/dialogflow-parameters.png)
*“item” parameter for “Add item request” intent*

But how does Dialogflow know in which point of the conversation we are, so only the right intent for our invocation phrase can be triggered? [Contexts](https://cloud.google.com/dialogflow/docs/contexts-overview) are the key to controlling the flow of the conversation and are used to define [follow-up intents](https://cloud.google.com/dialogflow/docs/contexts-follow-up-intents). When an intent is matched, any configured *output contexts* for that intent become active. While any contexts are active, Dialogflow will only match intents that are configured with *input contexts* that match the currently active contexts.

![Input and output contexts for “Add item request” intent](https://ismael.dev/static/articles/voxe/dialogflow-contexts.png)
*Input and output contexts for “Add item request” intent*

### Fulfillment

We’ve discussed so far how intents represent the input of the conversational action. But what about the output? In Dialogflow, intents have a built-in [response](https://cloud.google.com/dialogflow/docs/intents-responses) handler that can return **static responses** after the intent is matched. Different variations of this response can be specified, and one of them will be selected randomly.

![Responses to “Add item request” intent](https://ismael.dev/static/articles/voxe/dialogflow-responses.png)
*Responses to “Add item request” intent*

Most of the time we will need **dynamic responses** for our intents. The “Read list” intent is a good example. We need some code to access the database and read our list. For this to happen we need to use [fulfillment](https://cloud.google.com/dialogflow/docs/fulfillment-overview) for the intent. This tells Dialogflow to use an HTTP request to a webhook to determine the response.

In order to build the fulfillment for VOXE we’ve used the Node.js® [client library](https://github.com/actions-on-google/actions-on-google-nodejs) for Actions on Google. We have deployed our code to a [Cloud Function](https://firebase.google.com/docs/functions) using Firebase.

This snippet shows the implementation for the “Read list” intent handler:

```javascript
'use strict';

// Import the Dialogflow module and response creation dependencies
// from the Actions on Google client library.
const {
  dialogflow,
  BasicCard,
  List,
  SignIn,
  Suggestions,
} = require('actions-on-google');

// Import the firebase-functions package for deployment.
const functions = require('firebase-functions');

// Instantiate the Dialogflow client.
const app = dialogflow({
  debug: true,
  clientId: 'YOUR_CLIENT_ID_HERE',
});

// Init firebase
const admin = require('firebase-admin');

admin.initializeApp(functions.config().firebase);
const db = admin.firestore();
const auth = admin.auth();
const todosCollection = db.collection('todos');

// Middleware to store user uid
app.middleware(async (conv) => {
  const { email } = conv.user;
  const user = await auth.getUserByEmail(email);
  conv.user.storage.uid = user.uid;
});

// Handle the Read List intent
app.intent('read list', async (conv) => {
  const snapshot = await todosCollection
    .where('userId', '==', conv.user.storage.uid)
    .get()
    .catch((err) => {
      conv.close('Sorry. It seems we are having some technical difficulties.');
      console.error('Error reading Firestore', err);
    });

  if (!snapshot.empty) {
    let ssml = '<speak>This is your to-do list: <break time="1" />';
    snapshot.forEach((doc) => {
      ssml += `${doc.data().content}<break time="500ms" />, `;
    });
    ssml += '<break time="500ms" />Would you like me to do anything else?</speak>';
    conv.ask(ssml);
    conv.ask(new Suggestions('Yes', 'No'));
    conv.contexts.set('restart-input', 2);
  } else {
    conv.ask(`${conv.user.profile.payload.given_name}, your list is empty. Do you want to add a new item?`);
    conv.ask(new Suggestions('Yes', 'No'));
    conv.contexts.set('read-empty-list-followup', 2);
  }
});

// The rest of your intents handlers here
// ...

// Set the DialogflowApp object to handle the HTTPS POST request.
exports.dialogflowFirebaseFulfillment = functions.https.onRequest(app);
```

The full version of the fulfillment can be found [here](https://github.com/potatolondon/voxe/blob/master/functions/index.js).

### Multimodal interaction

The Google Assistant is available to users on multiple devices like smartphones, smart speakers and smart displays. This means we’re not limited to voice and text as the only method for input and output, but we can also use visuals. This is known as multimodal interaction, and takes full advantage of these devices capabilities. This makes it easy for users to quickly scan lists or make selections.

A good example of multimodal interaction happens when we need to specify an item to be removed from the list. One option is using voice (or text) to indicate the item. However, if we’re using a touchscreen device, it feels more natural to select it from the list directly.

![VOXE: Multimodal interaction to remove list item](https://ismael.dev/static/articles/voxe/multimodal.png)
*VOXE: Multimodal interaction to remove list item*

***

*In this two-part article we discussed how to build a cross-platform experience which includes voice assistants. The first part focused on exploring the technologies available and choosing a tech stack. In this second part we explained how we built a simple to-do list prototype app using NativeScript, Firebase and Actions on Google.*
