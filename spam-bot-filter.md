Managing multiple Gmail accounts over the years, I've relied on Google Drivefor file backups and Google Photos for photo storage. Despite keeping my primary inbox tidy, Gmail's categorization often leaves non-inbox emails unread, creating a digital clutter I've ignored. Recently, this oversight forced me to invest in a Google One subscription as my Drive reached its capacity. Google's policy of limiting email activity when storage is full prompted me to find a solution. Manually sorting through years of emails seemed daunting, leading me to explore the potential of AI assistance.

The Assistant API with function calling tools recently grabbed my attention. After writing a blog post exploring its potential applications, I came across an X (former Twitter) post where someone claimed to have used the API to organize their email inbox. This intrigued me — it seemed like a practical solution for my email management challenges. However, the post didn't provide any code, making me think that this could be a useful tool for many others sharing the same issue. So I decided to dive into the googleapi and combine it with the OpenAI assistant API, testing it out with a real-world use case.

## What will this code do?

The proposed code addresses the challenge of identifying and managing unread emails efficiently. It will analyze the latest unread emails in your inbox, categorizing them as spam/marketing or legitimate. Legitimate emails will be marked as read, while spam/marketing emails will be promptly deleted. This solution is designed for Gmail accounts, and users of other email providers will need to adapt the code to suit their specific APIs.

This proof of concept serves as a powerful tool, but I recommend exercising caution. Begin by testing it on a small dataset before scaling up to avoid potential data loss or other issues. I assume no responsibility for any consequences arising from the use of this code.

### Gmail CRUD operations

I've taken the initial code from their Node.js [quickstart](https://developers.google.com/gmail/api/quickstart/nodejs) guide, and added the functionality to mark as read, and delete emails. I won't go into much detail here since the code is pretty self explanatory.

I believe you'll have to create first a Google Cloud project. I already had one but it's fairly easy to create one. See steps [here](https://developers.google.com/workspace/guides/create-project).

### Adding required scopes to your app

While the [quickstart guide](https://developers.google.com/gmail/api/quickstart/nodejs) mentions the `readonly` scope, our solution requires additional scopes (`modify` and `delete`). During the OAuth consent screen configuration, you can add these permissions for effective email management. I initially added all scopes, ensuring functionality, but precise scope knowledge is recommended for production implementation.

![OAuth Consent Screen with Scopes](link-to-screenshot)

I honestly do enough research of which where the exact scopes that I needed, so I just added all of them which is obviously dangerous. For this like this why you should be careful when implementing this poc. If you know which ones are the exact ones please add them in the comments.

### The googleapi code

The code below is the one that I used to get the emails from my inbox. I've added some comments to explain what each part does.

```ts
// email.ts
const fs = require('fs').promises;
const path = require('path');
const process = require('process');
const { authenticate } = require('@google-cloud/local-auth');
const { google } = require('googleapis');

// this gives ALL permissions to the app - USE WITH CAUTION
// app should be configured before running this script in the OAuth consent screen -> Scopes for Google APIs
// If modifying these scopes, delete token.json.
const SCOPES = ['https://mail.google.com/'];

// The file token.json stores the user's access and refresh tokens, and is
// created automatically when the authorization flow completes for the first
// time.
// the credentials.json file is the one downloaded from the google docs tutorial
const TOKEN_PATH = path.join(process.cwd(), 'token.json');
const CREDENTIALS_PATH = path.join(
  process.cwd(),
  'email-cleaner/credentials.json'
);

// Load credentials or create new credentials if none exist
async function loadSavedCredentialsIfExist() {
  try {
    const content = await fs.readFile(TOKEN_PATH);
    const credentials = JSON.parse(content);
    return google.auth.fromJSON(credentials);
  } catch (err) {
    return null;
  }
}

async function saveCredentials(client) {
  const content = await fs.readFile(CREDENTIALS_PATH);
  const keys = JSON.parse(content);
  const key = keys.installed || keys.web;
  const payload = JSON.stringify({
    type: 'authorized_user',
    client_id: key.client_id,
    client_secret: key.client_secret,
    refresh_token: client.credentials.refresh_token,
  });
  await fs.writeFile(TOKEN_PATH, payload);
}

export async function authorize() {
  let client = await loadSavedCredentialsIfExist();
  if (client) {
    return client;
  }
  client = await authenticate({
    scopes: SCOPES,
    keyfilePath: CREDENTIALS_PATH,
  });
  if (client.credentials) {
    await saveCredentials(client);
  }
  return client;
}

// get message details
async function getMessage(auth, id) {
  const gmail = google.gmail({ version: 'v1', auth });
  return gmail.users.messages.get({
    userId: 'me',
    id,
  });
}

// get all unread emails from the inbox
export async function listUnreadMessages(auth, qty = 10) {
  const gmail = google.gmail({ version: 'v1', auth });
  const res = await gmail.users.messages.list({
    userId: 'me',
    maxResults: qty,
    q: 'is:unread',
  });
  const messages = res.data.messages;
  if (!messages || messages.length === 0) {
    console.log('No messages found.');
    return;
  }

  let messagesWithDetails = [];
  for await (const message of messages) {
    // get message data
    const res = await getMessage(auth, message.id);
    messagesWithDetails.push({ id: message.id, snippet: res.data.snippet });
  }

  return messagesWithDetails;
}

// delete one or more messages
export async function deleteMessages(auth, ids: string | string[]) {
  const gmail = google.gmail({ version: 'v1', auth });

  if (Array.isArray(ids)) {
    return gmail.users.messages.batchDelete({
      userId: 'me',
      ids,
    });
  }

  return gmail.users.messages.delete({
    userId: 'me',
    id: ids,
  });
}

// mark one or more messages as read
export async function markAsRead(auth, ids: string | string[]) {
  const gmail = google.gmail({ version: 'v1', auth });

  if (Array.isArray(ids)) {
    return gmail.users.messages.batchModify({
      userId: 'me',
      ids,
      requestBody: {
        removeLabelIds: ['UNREAD'],
      },
    });
  }

  return gmail.users.messages.modify({
    userId: 'me',
    id: ids,
    requestBody: {
      removeLabelIds: ['UNREAD'],
    },
  });
}
```

Again, the above code is very similar to the one in the [quickstart](https://developers.google.com/gmail/api/quickstart/nodejs) guide. I've added the `deleteMessages` and `markAsRead` functions only.

### Setting up the OpenAI Assistant API

The Assistant API, unveiled during OpenAI's recent devday, empowers developers to construct AI assistants within their applications. An Assistant, equipped with instructions, harnesses models, tools, and knowledge to respond to user queries. Presently, the API supports three types of tools: Code Interpreter, Retrieval, and Function Calling. OpenAI plans to release more built-in tools and eventually enable users to introduce their own tools on the platform.

For this Proof of Concept (POC), I leveraged the function calling tool, offering the flexibility to invoke any desired function with specific inputs provided by the assistant. This flexibility renders the limits of the Assistant API nearly boundless.

The assistant setup was directly facilitated through the OpenAI [Playground](https://platform.openai.com/assistants). I created a new assistant and incorporated the spam_message_filter function:

```json
{
  "messages": {
    "type": "array",
    "description": "The list of messages to filter",
    "items": {
      "type": "object",
      "properties": {
        "id": {
          "type": "string",
          "description": "The id of the message"
        },
        "snippet": {
          "type": "string",
          "description": "The snippet of the message"
        },
        "is_spam_or_marketing": {
          "type": "boolean",
          "description": "the decision"
        },
        "reason": {
          "type": "string",
          "description": "The reason why the message is spam"
        }
      },
      "required": ["id", "snippet", "reason", "is_spam_or_marketing"]
    }
  }
}
```

This function evaluates a list of messages and returns a categorized list with decisions on whether each message is spam, along with reasons for such classification.

To invoke this function, the assistant needs specific instructions. The provided instructions are straightforward:

> You expertly filter emails for spam using advanced techniques. Given email IDs and snippets, your role is to swiftly provide an object for spam_message_filter with the array of messages. Invoke this function without engaging in conversation, and upon completion, simply finish the run with 'success' or 'failure'.

These instructions ensure the assistant operates without seeking clarification, executing the function and returning the result promptly. The code_interpreter tool must be toggled on for proper functionality.

Note: While the Assistant API can be created directly from code (see [here](https://platform.openai.com/docs/assistants/overview?lang=node.js)), I opted to create the assistant from the playground and use the assigned assistant ID in the code to maintain consistency and avoid creating a new assistant with each invocation.

### The OpenAI Assistant API code

It's time to put everything together. The code below is the one that I used to get the emails from my inbox. I've added some comments to explain what each part does and I'll explain some important parts below.

```ts
import {
  authorize,
  deleteMessages,
  listUnreadMessages,
  markAsRead,
} from './email';
import { Thread } from 'openai/resources/beta/threads/threads';

// import the required dependencies
require('dotenv').config();
const OpenAI = require('openai');
const readline = require('readline').createInterface({
  input: process.stdin,
  output: process.stdout,
});

// Create a OpenAI connection
const apiKey = process.env.OPENAI_API_KEY;
const openai = new OpenAI({
  apiKey,
});

async function askQuestion(question: string) {
  return new Promise<string>((resolve, _reject) => {
    readline.question(question, (answer: string) => {
      resolve(`${answer}\n`);
    });
  });
}

type Message = {
  id: string;
  snippet: string;
  is_spam_or_marketing: boolean;
  reason: string;
};

async function initGmailAuth() {
  try {
    const auth = await authorize();
    return auth;
  } catch (error) {
    console.error(error);
    throw error;
  }
}

async function spamMessageFilter(messages: Message[], auth) {
  const delIds = messages
    .filter((message) => message.is_spam_or_marketing)
    .map((message) => message.id);

  // delete the messages
  deleteMessages(auth, delIds)
    .then(() => console.log('messages deleted', delIds))
    .catch(() => {
      // throw
      console.log('error deleting messages with ids', delIds);
    });

  const readIds = messages
    .filter((message) => !message.is_spam_or_marketing)
    .map((message) => message.id);

  // mark as read
  markAsRead(auth, readIds)
    .then(() => console.log('messages marked as read', readIds))
    .catch(() => {
      // throw
      throw new Error('error marking messages as read with ids', readIds);
    });
}

async function fetchLatestUnreadEmails(quantity: number = 10, auth) {
  return await listUnreadMessages(auth, quantity);
}

async function main() {
  // use the assistant created using the OpenAI playground
  const assistantId = process.env.OPENAI_SPAM_FILTER_ASSISTANT_ID;

  if (!assistantId) {
    throw new Error('OPENAI_SPAM_FILTER_ASSISTANT_ID not found');
  }
  const assistant = await openai.beta.assistants.retrieve(assistantId);

  // Log the first greeting
  console.log(
    "\nHello there, I'm your personal spam filter. I'll help you filter your emails.\n"
  );

  // this flag will be used prompt the user to continue or not
  let keepCleaning = true;
  // once a thread has been created, we can use the thread id to continue the conversation
  let threadId: string;
  while (keepCleaning) {
    // fetch the latest unread emails
    const gmailAuth = await initGmailAuth();

    // invoke the google api to fetch the latest unread emails
    const messages = await fetchLatestUnreadEmails(10, gmailAuth);
    if (Array.isArray(messages) && messages.length === 0) {
      console.log('Failed to fetch messages');
      return;
    }

    // create a thread
    let thread: Thread;
    if (!threadId) {
      thread = await openai.beta.threads.create();
    } else {
      thread = await openai.beta.threads.retrieve(threadId);
    }
    threadId = thread.id;

    console.log('starting thread with id: ', threadId);

    const messagesAsString = JSON.stringify(messages);
    await openai.beta.threads.messages.create(thread.id, {
      role: 'user',
      // todo?: when not passed as markdown, the assistant will not be able to parse the json
      content: `\`\`\`json\n${messagesAsString}\n\`\`\``,
    });
    console.log('thinking...');

    // use runs to wait for the assistant response and then retrieve it
    const run = await openai.beta.threads.runs.create(thread.id, {
      assistant_id: assistant.id,
    });

    let actualRun = await openai.beta.threads.runs.retrieve(thread.id, run.id);

    // polling mechanism to see if actualRun is completed
    // and we're ready to retrieve the assistant response
    // this should be made more robust.
    while (
      actualRun.status === 'queued' ||
      actualRun.status === 'in_progress' ||
      actualRun.status === 'requires_action'
    ) {
      // requires_action means that the assistant is waiting for the functions to be invoked
      // the model should provide a json object with the shape of the json objects above
      if (actualRun.status === 'requires_action') {
        // extra single tool call
        const toolCall =
          actualRun.required_action?.submit_tool_outputs?.tool_calls[0];

        const name = toolCall?.function.name;

        // parse the response from the assistant
        const args = JSON.parse(toolCall?.function?.arguments || '{}');
        const messages = args.messages;

        if (name === 'spam_message_filter') {
          await spamMessageFilter(messages, gmailAuth);
        } else {
          console.log('unknown function');
        }

        // we must submit the tool outputs to the run to continue
        await openai.beta.threads.runs.submitToolOutputs(thread.id, run.id, {
          tool_outputs: [
            {
              tool_call_id: toolCall?.id,
              output: JSON.stringify({ success: true }),
            },
          ],
        });
      }
      // keep polling until the run is completed
      await new Promise((resolve) => setTimeout(resolve, 2000));
      actualRun = await openai.beta.threads.runs.retrieve(thread.id, run.id);
    }

    // Get the last assistant message from the messages array
    const assisstantMessages = await openai.beta.threads.messages.list(
      thread.id
    );

    // Find the last message for the current run
    const lastMessageForRun = assisstantMessages.data
      .filter(
        (message) => message.run_id === run.id && message.role === 'assistant'
      )
      .pop();

    // If an assistant message is found, console.log() it
    if (lastMessageForRun) {
      // aparently this is not correctly typed
      // content returns an of objects do contain a text object
      const messageValue = lastMessageForRun.content[0] as {
        text: { value: string };
      };

      console.log(messageValue?.text?.value);
    }

    // ask if the user wants to continue
    const answer = await askQuestion('Do you want to continue? (y/n) ');
    if (answer.startsWith('n')) {
      keepCleaning = false;
    }
  }
}

main().catch(console.error);
```

There are some key parts that I will explain next. If you want to understand better the rest of the code, please you can take a look at my previous [blog post](https://dev.to/esponges/build-the-new-openai-assistant-with-function-calling-52f5).

The `requires_action` status means that the assistant has already processed the input and is waiting for the functions to be invoked. The model should provide a json object with the shape of the json objects above.ºº  The `spam_message_filter` function is the one that we created in the OpenAI playground. The `spamMessageFilter` function is the one that invokes the google api to delete and mark as read the emails.

```ts
if (actualRun.status === 'requires_action') {
  // extra single tool call
  const toolCall =
    actualRun.required_action?.submit_tool_outputs?.tool_calls[0];

  const name = toolCall?.function.name;

  // parse the response from the assistant
  const args = JSON.parse(toolCall?.function?.arguments || '{}');
  const messages = args.messages;

  // ...
}
```

Having the necessary information, we can invoke the `spamMessageFilter` function. This is a function that is created by us and it can do anything we want. In this case, using the methods from the `email.ts` file we'll first delete the spam emails and then mark the remaining ones as read.

```ts
async function spamMessageFilter(messages: Message[], auth) {
  const delIds = messages
    .filter((message) => message.is_spam_or_marketing)
    .map((message) => message.id);

  // delete the messages
  deleteMessages(auth, delIds)
    .then(() => console.log('messages deleted', delIds))
    .catch(() => {
      // throw
      throw new Error('error deleting messages');
    });

  const readIds = messages
    .filter((message) => !message.is_spam_or_marketing)
    .map((message) => message.id);

  // mark as read
  markAsRead(auth, readIds)
    .then(() => console.log('messages marked as read', readIds))
    .catch(() => {
      // throw
      throw new Error('error marking messages as read');
    });
}
```

Finally, after submitting a `success` response to the assistant, we can continue looping and asking the user if they want to continue cleaning their inbox.

```ts
  const answer = await askQuestion('Do you want to continue? (y/n) ');
  if (answer.startsWith('n')) {
    keepCleaning = false;
  }
```

This should be enough for this PoC to work. You can try increasing the number of emails to be processed by changing the `quantity` argument from the `fetchLatestUnreadEmails` function. 

Once you're sure that everything is working as expected, you could decrease the input tokens provided to the assistant, probably removing the `reason` and `snippet` properties from the `Message` type. This will reduce the cost of each call to the assistant.


