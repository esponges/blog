How to Build a Contextual Chatbot with LangChain and PostgreSQL + Drizzle ORM

Something that I've always disliked of using ChatGPT or Bard for understanding a document is that I have always to copy and paste chunks of the document into my actual conversation. This is not only annoying but also time consuming. I've always wanted to have a chatbot that could understand the context of the conversation and the document that I'm reading and be able to answer my questions without having to copy and paste anything.

The great things of contextual chats is that you get rid of the need of having to copy and paste anything. You can just ask your chatbot a question and it will answer you. This is great for reading documents and having a conversation with your chatbot at the same time. This is not only for people that would like to understand more about an specific topic but also for websites that would like to have a chatbot that can answer about their ONLY products or services in an efficient and scalable way.

Fortunately with the rise of technologies like the OpenAI API, Lanchain which is a library made to make developing applications powered by language models easier and PostgreSQL and Drizzle which is the raising star of ORMs, we can now make this possible with a few lines of code.

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

Please remember that I'm using the NextJS routes api so you'll have to adapt this code to your framework of choice.

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

//create chain for conversational AI
const chain = await makeChain(HNSWStore);

//Ask a question using chat history
// OpenAI recommends replacing newlines with spaces for best results
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