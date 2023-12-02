
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

