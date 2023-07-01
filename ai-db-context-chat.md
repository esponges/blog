How to Build a Contextual Chatbot with LangChain and PostgreSQL + Drizzle ORM

Something that I've always disliked of using ChatGPT or Bard for understanding a document is that I have always to copy and paste chunks of the document into my actual conversation. This is not only annoying but also time consuming. I've always wanted to have a chatbot that could understand the context of the conversation and the document that I'm reading and be able to answer my questions without having to copy and paste anything.

The great things of contextual chats is that you get rid of the need of having to copy and paste anything. You can just ask your chatbot a question and it will answer you. This is great for reading documents and having a conversation with your chatbot at the same time. This is not only for people that would like to understand more about an specific topic but also for websites that would like to have a chatbot that can answer about their ONLY products or services in an efficient and scalable way.

Fortunately with the rise of technologies like the OpenAI API, Lanchain which is a library made to make developing applications powered by language models easier and PostgreSQL and Drizzle which is the raising star of ORMs, we can now make this possible with a few lines of code.

For this tutorial you should be familiar with the basics of PostgreSQL and Drizzle (or similar ORMs like PRisma) and probably NextJS but you can adapt this tutorial to any other framework since we'll focus on the backend only

I don't want to make this post too long adding frontend code. I used this project as a base for the frontend and adapted the backend to support the upload and query of documents to have it working with a contextually aware chatbot.

Our Drizzle model will be the following

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
```

Then we'll have a route that will upload a document to our database only if this document doesn't exist already. We don't want repeated documents in our database for obvious reasons.

From the client the request will be something like this:

```ts
const formData = new FormData();

// some existing document of the type of File
formData.append('file', file);

const response = await fetch('/api/upload', {
  method: 'POST',
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

back in the api/upload.ts file we'll check the `document.length` to see if the document already exists

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
        metadata: JSON.stringify(doc.metadata),
        pageContent: doc.pageContent,
        langChainDocsId: newDocId,
      }))
    );
  });
};
```
