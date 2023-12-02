
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

```ts
const fs = require('fs').promises;
const path = require('path');
const process = require('process');
const { authenticate } = require('@google-cloud/local-auth');
const { google } = require('googleapis');

