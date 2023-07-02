Have you ever wanted to have a chatbot that could understand the context of a conversation and a document? For example, imagine you're reading a document about how to build a chatbot, and you have a question about a specific step. You could ask your chatbot the question, and it would be able to answer you without you having to copy and paste anything.

This is what contextual chats are all about. They allow you to have a natural conversation with a chatbot, even if the chatbot is not familiar with the topic of the conversation.

Contextual chats are made possible by a combination of technologies, including LangChain, PostgreSQL, and Drizzle.

- LangChain is a framework that makes it easy to interact with language models.
- PostgreSQL is a database that can be used to store documents.
- Drizzle is an ORM that can be used to query documents in PostgreSQL.

In this tutorial, we will show you how to build a contextual chatbot using these technologies. We will start by creating a simple chatbot that can answer questions about documents. Then, we will show you how to use LangChain to interact with a language model, PostgreSQL to store documents, and Drizzle to query documents.

By the end of this tutorial, you will have an idea of how to build a working contextual chatbot that can answer questions about documents. You will also have a better understanding of how these technologies can be used to build contextual chatbots.

In my case I wanted the interface to accept a PDF file and upload it to the database and it looked like this:

![chatbot preview](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/k5512r07iwb77wdak8w1.png)

---

With no further ado, let's get started!

For this tutorial you should be familiar with Typescript, PostgreSQL and Drizzle (or similar ORMs like PRisma) and probably NextJS but you can adapt this tutorial to any other framework since we'll focus on the backend only

I don't want to make this post too long adding frontend code. I used this project as a base for the frontend and adapted the backend to support the upload and query of documents to have it working with a contextually aware chatbot.

Our Drizzle model will contain a parent document `LangChainDocs` and a child document `Docs`. The parent document will contain the name of the document and the child document will contain the metadata and the content of the document.

```ts
// your-drizzle-model.ts
import { relations } from 'drizzle-orm';
import { pgTable, text, varchar } from 'drizzle-orm/pg-core';

export const langChainDocs = pgTable('LangChainDocs', {
  id: varchar('id').primaryKey(),
  createdAt: text('createdAt'),
  name: text('name'),
  nameSpace: text('nameSpace'),
});

export const langChainDocRelations = relations(langChainDocs, ({ many }) => ({
  docs: many(docs),
}));

export const docs = pgTable('Docs', {
  id: varchar('id').primaryKey(),
  createdAt: text('createdAt'),
  metadata: text('metadata'),
  pageContent: text('pageContent'),
  name: text('name'),
  langChainDocsId: text('langChainDocsId'),
});

export const docsRelations = relations(docs, ({ one }) => ({
  langChainDocs: one(langChainDocs, {
    fields: [docs.langChainDocsId],
    references: [langChainDocs.id],
  }),
}));

// your-drizzle-db.ts
export const drizzleDb = drizzle(client, { schema });
// more of how to setup drizzle in https://drizzle.dev/docs/#getting-started
```

Then we'll have a route that will upload a document to our database only if this document doesn't exist already. We don't want repeated documents in our database for obvious reasons.

From the client the request will be something like this:

```ts
const formData = new FormData();

// some existing document of the type of File
formData.append('file', file);

const response = await fetch('/api/upload', {
  method: 'POST',
  // you'll probably have to add the multipart/form-data header
  body: formData,
});
```

Then in the backend we'll parse the file with the `multiparty` library, this will give us the file name and the path of the file in the server local storage.

```ts
// api/upload.ts
import { Form } from 'multiparty';

export default async function handler(
  req: ApFDataRequest,
  res: NextApiResponse,
) {
  const form = new Form();
  const formData = await new Promise<FData>((resolve, reject) => {
    form.parse(req, (err, fields, files) => {
      if (err) {
        reject(err);
        return;
      }

      const file = files.file[0];
      resolve({ file });
    });
  });

  const fileName = formData.file.originalFilename;
  const filePath = formData.file.path;
```

Once having the parsed file we'll query the database to see if the document already exists. If it doesn't exist we'll upload it to the database and we'll return the document name to the client.

```ts
// api/upload.ts
import { getExistingDocs } from 'your-backend';

const DBDocs = await getExistingDocs(fileName);

// somewhere in your backend or directly in the api/upload.ts file
export const getExistingDocs = async (fileName: string) => {
  const document = await drizzleDb.query.langChainDocs.findMany({
    where: eq(schema.langChainDocs.name, fileName),
    with: {
      docs: true,
    },
  });

  return document;
};
```

back in the api/upload.ts file we'll check the `document.length` to see if the document already exists in the database.

```ts
const DBDocs = await getExistingDocs(fileName);
const fileExistsInDB = DBDocs.length > 0;

if (!fileExistsInDB) {
  // upload the document to the database
} else {
  // return the document name to the client
}
```

Now it's time to parse the file in a way that Langchain can understand and upload it to the database.

```ts
// somewhere in your backend or directly in the api/upload.ts file
import { Document } from 'langchain/document';
import { PDFLoader } from 'langchain/document_loaders/fs/pdf';
import { RecursiveCharacterTextSplitter } from 'langchain/text_splitter';

const getPdfText = async (
  // the path of the file in the server local storage
  filePath: string
): Promise<Document<Record<string, any>>[]> => {
  const loader = new PDFLoader(filePath);

  const pdf = await loader.load();

  // split into chunks
  const textSplitter = new RecursiveCharacterTextSplitter({
    chunkSize: 1000,
    chunkOverlap: 200,
  });

  // this outputs an array of the type of Document objects
  // https://docs.langchain.com/docs/components/schema/document
  const docs = await textSplitter.splitDocuments(pdf);

  return docs;
};
```

with the parsed document we can now upload it to the database using Drizzle ORM

```ts
const drizzleInsertDocs = async (
  docsToUpload: Document[],
  fileName: string
) => {
  await drizzleDb.transaction(async () => {
    const newDocId = randomUUID();

    await drizzleDb
      .insert(langChainDocs)
      .values({
        id: newDocId,
        name: fileName,
        nameSpace: fileName,
      })
      .returning();

    await drizzleDb.insert(docs).values(
      docsToUpload.map((doc) => ({
        id: randomUUID(),
        name: fileName,
        // metadata is a JSON object thus we need to stringify it
        metadata: JSON.stringify(doc.metadata),
        pageContent: doc.pageContent,
        langChainDocsId: newDocId,
      }))
    );
  });
};
```

and they are invoked like this:

```ts
export const langchainUploadDocs = async (
  filePath: string,
  fileName: string
) => {
  const docs = await getPdfText(filePath);

  await drizzleInsertDocs(docs, fileName);
};
```

all together the api/upload.ts file will look like this:

```ts
import type { NextApiRequest, NextApiResponse } from 'next';
import { Form } from 'multiparty';

import { langchainUploadDocs } from '@/utils/langchain';
import { getErrorMessage } from '@/utils/misc';
import { getExistingDocs } from '@/utils/drizzle';

export const config = {
  api: {
    bodyParser: false,
  },
};

interface FData {
  file: {
    fieldName: string;
    originalFilename: string;
    path: string;
    headers: {
      [key: string]: string;
    };
    size: number;
  };
}

interface ApFDataRequest extends NextApiRequest {
  body: FData;
}

export type UploadResponse = {
  fileExistsInDB: boolean;
  nameSpace: string;
};

export default async function handler(
  req: ApFDataRequest,
  res: NextApiResponse
) {
  if (req.method !== 'POST') {
    res.status(405).json({ error: 'Method not allowed' });
    return;
  }
  const form = new Form();
  const formData = await new Promise<FData>((resolve, reject) => {
    form.parse(req, (err, fields, files) => {
      if (err) {
        reject(err);
        return;
      }

      const file = files.file[0];
      resolve({ file });
    });
  });

  const fileName = formData.file.originalFilename;
  const filePath = formData.file.path;

  try {
    const DBDocs = await getExistingDocs(fileName);
    const fileExistsInDB = DBDocs.length > 0;

    if (!fileExistsInDB) {
      try {
        await langchainUploadDocs(filePath, fileName);
      } catch (error) {
        const errMsg = getErrorMessage(error);
        res.status(500).json({ error: errMsg });
        return;
      }
    }

    const resData: UploadResponse = {
      fileExistsInDB: !!fileExistsInDB,
      nameSpace: fileName,
    };

    res.status(200).json(resData);
  } catch (error) {
    const errMsg = getErrorMessage(error);
    res.status(500).json({ error: errMsg });
    return;
  }
}
```

> Please remember that I'm using the NextJS routes api so you'll have to adapt this code to your framework of choice.

Great, we now have a file uploaded to our database. Now it's time to query the database to get the document and the metadata of the document and use it to have a conversation with our chatbot using the tools that Langchain provides us.

Back in the client, we've received a response from the server with the name of the document. We'll use this name to call the `api/chat` route for the actual contextual conversation

The client will send to the server a request like this:

```ts
// shape of the request
interface ReqBody {
  question: string;
  history: Array<Array<string>>;
  nameSpace: string;
}

// 1st iteration of the conversation
// question: 'Please give me an overview of the document',
const req = {
  question: 'Please give me an overview of the document',
  history: [],
  nameSpace: 'tasty-cakes.pdf',
};

// 2nd iteration of the conversation
const req = {
  question: 'Do they have chocolate?',
  history: [
    [
      'Please give me an overview of the document',
      'The document is about tasty cakes',
    ],
  ],
  nameSpace: 'tasty-cakes.pdf',
};

// 3rd iteration of the conversation
const req = {
  question: 'Do they have vanilla?',
  history: [
    [
      'Please give me an overview of the document',
      'The document is about tasty cakes',
    ],
    ['Do they have chocolate?', 'Yes, they have chocolate'],
  ],
  nameSpace: 'tasty-cakes.pdf',
};

// and so on...
```

Which will be sent to the server like this:

```ts
const response = await fetch('/api/chat', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
  },
  body: JSON.stringify(req),
});
```

Then back in our `api/chat.ts` file we'll parse the request and query the database to get the document and the metadata of the document.

```ts
// use the nameSpace to query the database with our getExistingDocs function
const DBDocs = await getExistingDocs(nameSpace);

// we must arrange them in a way that Langchain can understand using the Document class
import { Document } from 'langchain/document';

const documents = sqlDocs[0].docs.map(
  (doc) =>
    new Document({
      metadata: JSON.parse(doc.metadata as string),
      pageContent: doc.pageContent as string,
    })
);
```

We've also have to arrange the chat history in a way that Langchain can understand:

```ts
// api/chat.ts
import {
  AIChatMessage,
  BaseChatMessage,
  HumanChatMessage,
} from 'langchain/schema';

const chatHistory: BaseChatMessage[] = [];
history?.forEach((_, idx) => {
  // first message is always human message
  chatHistory.push(new HumanChatMessage(history[idx][0]));
  // second message is always AI response
  chatHistory.push(new AIChatMessage(history[idx][1]));
});
```

We'll use the HNSWLib library to embed locally the documents in a vector space. This will allow us to query the database for the most similar document to the question that we're asking.

```ts
// api/chat.ts
const HNSWStore = await HNSWLib.fromDocuments(
  documents,
  new OpenAIEmbeddings()
);
```

The we'll create something called chain (add small description):

```ts
// somewhere in your backend or directly in the api/chat.ts file
import { OpenAI } from 'langchain/llms/openai';
import { ConversationalRetrievalQAChain } from 'langchain/chains';
import { VectorStore } from 'langchain/dist/vectorstores/base';

const CONDENSE_PROMPT = `Given the following conversation and a follow up question, rephrase the follow up question to be a standalone question.

Chat History:
{chat_history}
Follow Up Input: {question}
Standalone question:`;

const QA_PROMPT = `You are a helpful AI assistant. Use the following pieces of context to answer the question at the end.
If you don't know the answer, just say you don't know. DO NOT try to make up an answer.
If the question is not related to the context, politely respond that you are tuned to only answer questions that are related to the context.

{context}

Question: {question}
Helpful answer in markdown:`;

export const makeChain = async (vectorStore: VectorStore) => {
  const model = new OpenAI({
    temperature: 0.9, // increase temepreature to get more creative answers
    modelName: 'gpt-3.5-turbo', //change this to gpt-4 if you have access
    openAIApiKey: process.env.OPENAI_API_KEY,
  });

  return ConversationalRetrievalQAChain.fromLLM(
    model,
    vectorStore.asRetriever(),
    {
      qaTemplate: QA_PROMPT,
      questionGeneratorChainOptions: { template: CONDENSE_PROMPT },
      returnSourceDocuments: true, // optional
    }
  );
};
```

and we invoke the chain like this:

```ts
const HNSWStore = await HNSWLib.fromDocuments(
  documents,
  new OpenAIEmbeddings()
);

const chain = await makeChain(HNSWStore);

// Sanitize the question since OpenAI recommends replacing newlines with spaces for best results
const sanitizedQuestion = question.trim().replaceAll('\n', ' ');
const response = await chain.call({
  question: sanitizedQuestion,
  chat_history: chatHistory || [],
});
```

The response will return a text string and an array of Document objects make of which chunk of the document the LLM used to answer the question.

```ts
type Response = {
  // 'The document is about tasty cakes'
  answer: string;
  sourceDocuments: Document[];
};
```

That was it. Apologies for the long post but I wanted to make sure that I covered all the steps to make this work.

Here's the [source code] (https://github.com/esponges/gpt-langchain-upload-doc-chatbot) of a working example in case you want to play with, check more details and -hopefully- improve it!

## Conclusion

In this tutorial, we have shown you how to build a contextual chatbot using LangChain, PostgreSQL, and Drizzle. We started by creating a simple chatbot that could answer questions about documents. Then, we showed you how to use LangChain to interact with a language model, PostgreSQL to store documents, and Drizzle to query documents.

By the end of this tutorial, you have a working contextual chatbot that can answer questions about documents. You also have a better understanding of how these technologies can be used to build contextual chatbots.

Here are some of the next steps that you can take:

Improve the chatbot's ability to answer questions. You can do this by training the language model on a larger dataset of documents.
Add more features to the chatbot. For example, you could add the ability to generate summaries of documents, or the ability to answer open-ended questions.
Deploy the chatbot to a production environment. This would allow you to share the chatbot with others and gather feedback.

I hope you enjoyed this tutorial!

### Bonus

Query and update with Prism ORM

```ts
// schema.prisma.ts

model LangChainDocs {
  id        String   @id @default(uuid())
  createdAt DateTime @default(now())
  name      String
  nameSpace String
  docs      Docs[]
}

model Docs {
  id              String        @id @default(uuid())
  createdAt       DateTime      @default(now())
  metadata        String // json string
  pageContent     String
  name            String
  docs            LangChainDocs @relation(fields: [langChainDocsId], references: [id])
  langChainDocsId String
}

// for uploading documents to the database
const prismaInsertDocs = async (docsToUpload: Document[], fileName: string) => {
  await prisma.langChainDocs.create({
    data: {
      name: fileName,
      nameSpace: fileName,
      docs: {
        create: docsToUpload.map((doc) => ({
          name: fileName,
          metadata: JSON.stringify(doc.metadata),
          pageContent: doc.pageContent,
        })),
      },
    },
  });
};

// for querying the database
export const getDocumentsFromDB = async (fileName: string) => {
  const docs = await prisma.langChainDocs.findFirst({
    where: {
      name: fileName,
    },
    include: {
      docs: true,
    },
  });

  return docs;
};
```

Sources:

As Isaac Newton said: "If I have seen further it is by standing on the shoulders of Giants".

Most of the frontend has been kept as is from [the original project] (https://github.com/mayooear/gpt4-pdf-chatbot-langchain) that I used to get started with [Langchain] (https://js.langchain.com/docs/) and made a number of changes to make it work using PostgreSQL and [Drizzle] (https://orm.drizzle.team/docs/quick-start) ORM.
