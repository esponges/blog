Just a week has passed since the OpenAI Dev Conf 2023, and a surprising revelation unfolded: The Assistant's API. Unveiled in the latest OpenAI blog [post](https://openai.com/blog/new-models-and-developer-products-announced-at-devday), the Assistants API marks a significant stride in empowering developers to craft agent-like experiences within their applications.

In their own words:

> Today, we’re releasing the Assistants API, our first step towards helping developers build agent-like experiences within their own applications. An assistant is a purpose-built AI that has specific instructions, leverages extra knowledge, and can call models and tools to perform tasks.

So, what does this mean for us? In essence, it means we now have the capability to construct our own AI assistants using the OpenAI API. This empowers us to harness the prowess of the GPT-4 model to execute tasks and answer questions with additional knowledge and context that we provide through uploaded documents (with no need for third-party involvement!), a Python code interpreter operating in a sandboxed environment, and a feature that truly caught my attention: __function calling__.

While functions themselves aren't novel, it's the way they've been implemented that truly stands out. In the past, calling a function meant uncertainty about the model's return, necessitating post-processing for the desired result — and even then, success wasn't guaranteed. Now, we can specify the desired function output, and the model will strive to provide a response aligning with the provided schema.

This newfound tool grants us immense flexibility and power, enabling our assistant to perform _virtually any task__ — from sending emails to making calls and querying databases. The possibilities are limitless.

While the available documents and examples are still somewhat scarce, my curiosity led me to dive in and explore the potential. In this journey, I set out to build a simple math assistant, serving as a proof of concept for implementing function calls within the new Assistant API in Node.js.

For this example we'll initially create a simple quiz to test function calling and then we'll be able to keep the conversation going with the assistant to ask more questions while keeping the context of the conversation.

### The implementation

For this example I've used the cookbook example provided by OpenAI and the command line strategy from this post and I've tweaked it a bit to make it more interactive.

We'll just need a few basic things:

- An OpenAI API key
- A Node.js environment

We'll just need the `openai` and `dotenv` packages to get started:

```bash
npm install openai dotenv
```

Declare your API key as an environment variable: in your .env
```bash
OPENAI_API_KEY=your-api-key
```

Then, we can get started with the code:

```ts
// import the required dependencies
require('dotenv').config();
const OpenAI = require('openai');
const readline = require('readline').createInterface({
  input: process.stdin,
  output: process.stdout,
});

// Create a OpenAI connection
const secretKey = process.env.OPENAI_API_KEY;
const openai = new OpenAI({
  apiKey: secretKey,
});
```

We'll create a method and `readline`to wait for user input

```ts
async function askRLineQuestion(question: string) {
  return new Promise<string>((resolve, _reject) => {
    readline.question(question, (answer: string) => {
      resolve(`${answer}\n`);
    });
  });
}
```

Now we'll create a `main` function to run our program. We'll start by creating the assistant. 

This assistant will require the `code_interpreter` and `function`,  tools, and we'll use the `gpt-4-1106-preview` model. I've experimented with the `gpt-3.5-turbo-1106` model, but it doesn't seem to work as well as the `gpt-4-1106-preview` model.

I've abastracted the function call into a `quizJson` function that will return a JSON object with the quiz questions and answers just to make it easier to read.

```ts
const quizJson = {
  name: "display_quiz",
  description:
    "Displays a quiz to the student, and returns the student's response. A single quiz can have multiple questions.",
  parameters: {
    type: "object",
    properties: {
      title: { type: "string" },
      questions: {
        type: "array",
        description:
          "An array of questions, each with a title and potentially options (if multiple choice).",
        items: {
          type: "object",
          properties: {
            question_text: { type: "string" },
            question_type: {
              type: "string",
              enum: ["MULTIPLE_CHOICE", "FREE_RESPONSE"],
            },
            choices: { type: "array", items: { type: "string" } },
          },
          required: ["question_text"],
        },
      },
    },
    required: ["title", "questions"],
  },
};

async function main() {
  try {
    const assistant = await openai.beta.assistants.create({
      name: "Math Tutor",
      instructions:
        "You are a personal math tutor. Answer questions briefly, in a sentence or less.",
      tools: [
        { type: "code_interpreter" },
        {
          type: "function",
          function: quizJson,
        },
      ],
      // will work much better with the new model
      model: "gpt-4-1106-preview",
      // model: "gpt-3.5-turbo-1106",
    });

    // Log a first greeting
    console.log(
      "\nHello there, I'm Fernando's personal Math assistant. We'll start with a small quiz.\n",
    );
```

Once the assitant is created we'll create a _thread_, this will mantain the state of our conversation so we don't have to provide the context every time we ask a question. Remember that the model is stateless, so we need to provide the context every time we ask a question.

```ts
const thread = await openai.beta.threads.create();
```

To enable the application to run repeatedly, we'll employ a `while` loop. This loop will assess user input after each question to determine whether the user intends to continue or not. We'll also have a `isQuizAnswered` variable to keep track of the quiz state.

```ts
    // main method
    // create the assistant and the thread as mentioned above

    let continueConversation = true;
    let isQuizAnswered = false;

    while (continueConversation) {
      // logic

      // once done with the question-answer check if the user wants to continue
      const continueAsking = await askRLineQuestion(
        "Do you want to keep having a conversation? (yes/no) ",
      );

      continueConversation = continueAsking.toLowerCase() === "yes";

      // If the continueConversation state is falsy show an ending message
      if (!continueConversation) {
        console.log("Alrighty then, I hope you learned something!\n");
      }
    }
```

The logic for the question-answer process will be as follows:

```ts
    while (continueConversation) {
      // first ask the question and wait for the answer
      // we'll initiate with a quiz and then we'll keep the conversation going
      const userQuestion = isQuizAnswered
        ? await askRLineQuestion("You next question to the model: \n")
        // this will make the model  build a quiz using our provided function
        : "Make a quiz with 2 questions: One open ended, one multiple choice" +
          "Then, give me feedback for the responses.";

      // Pass in the user question into the existing thread
      await openai.beta.threads.messages.create(thread.id, {
        role: "user",
        content: userQuestion,
      });

      // Use runs to wait for the assistant response and then retrieve it
      // Creating a run will indicate to an assistant that it should start looking at the messages in a thread and take action by calling tools or the model.
      const run = await openai.beta.threads.runs.create(thread.id, {
        assistant_id: assistant.id,
      });

      // then retrieve the actual run
      let actualRun = await openai.beta.threads.runs.retrieve(
        // use the thread created earlier
        thread.id,
        run.id,
      );
```

What comes next is a polling strategy to wait for the model to finish processing the response. This is a bit of a hack, but it works for now. We'll wait for the model to finish processing the response and then we'll retrieve it.

The expected cycle is as follows:

- Previous to the quiz: The model will return a `queued` then an `in_progress` status while it's processing the response
- Once the `tool_calls` are added for later use, the model will return a `requires_action` status. __Here's where we'll actually execute the function__.
- Once the function is executed, we'll submit the tool outputs to the run to continue the conversation.
- Finally, the model will return a `completed` status and we'll retrieve the response.
- If the user wants to continue the conversation, we'll repeat the process but this time we'll skip the quiz and we'll just ask the user for a question.

```ts
      while (
        actualRun.status === "queued" ||
        actualRun.status === "in_progress" ||
        actualRun.status === "requires_action"
      ) {
        // requires_action means that the assistant is waiting for the functions to be added
        if (actualRun.status === "requires_action") {
          // extra single tool call
          const toolCall =
            actualRun.required_action?.submit_tool_outputs?.tool_calls[0];

          const name = toolCall?.function.name;

          const args = JSON.parse(toolCall?.function?.arguments || "{}");
          const questions = args.questions;

          const responses = await displayQuiz(name || "cool quiz", questions);

          // toggle flag that sets initial quiz
          isQuizAnswered = true;

          // we must submit the tool outputs to the run to continue
          await openai.beta.threads.runs.submitToolOutputs(
            thread.id,
            run.id,
            {
              tool_outputs: [
                {
                  tool_call_id: toolCall?.id,
                  output: JSON.stringify(responses),
                },
              ],
            },
          );
        }
        // keep polling until the run is completed
        await new Promise((resolve) => setTimeout(resolve, 2000));
        actualRun = await openai.beta.threads.runs.retrieve(thread.id, run.id);
      }
```

By this point we should have gotten a reponse from the model, so we'll display it to the user and then we'll ask if they want to continue the conversation.

```ts
      // once the run is completed, display the response
      console.log(actualRun.results[0].assistant_messages[0].content);

      // then ask if the user wants to continue
      const continueAsking = await askRLineQuestion(
        "Do you want to keep having a conversation? (yes/no) ",
      );

      continueConversation = continueAsking.toLowerCase() === "yes";

      // If the continueConversation state is falsy show an ending message
      if (!continueConversation) {
        console.log("Alrighty then, I hope you learned something!\n");
      }
    }
```

Finally, we'll add the `displayQuiz` function. This function will take the name of the quiz and the questions and it will display them to the user. It will then wait for the user to answer the questions and then it will return the responses.

```ts
      // Get the last assistant message from the messages array
      const messages = await openai.beta.threads.messages.list(thread.id);

      // Find the last message for the current run
      const lastMessageForRun = messages.data
        .filter(
          (message) =>
            message.run_id === run.id && message.role === "assistant",
        )
        .pop();

      // If an assistant message is found, console.log() it
      if (lastMessageForRun) {
        // aparently the `content` array is not correctly typed
        // content returns an of objects do contain a text object
        const messageValue = lastMessageForRun.content[0] as {
          text: { value: string };
        };

        console.log(`${messageValue?.text?.value} \n`);
      }
```

To keep the conversation we have the option to again read the user input which will allows use to toggle off —if necessary— the `continueConversation` flag or repeat the process in a conversational manner.

```ts
      // then ask if the user wants to continue
      const continueAsking = await askRLineQuestion(
        "Do you want to keep having a conversation? (yes/no) ",
      );

      continueConversation = continueAsking.toLowerCase() === "yes";

      // If the continueConversation state is falsy show an ending message
      if (!continueConversation) {
        console.log("Alrighty then, I hope you learned something!\n");
      }
    }
```

Don't forget to close the `readline` interface.

```ts
    readline.close();
  } catch (error) {
    console.error(error);
  }
}
```

Finally, we'll call the `main` function to run the program.

```ts
// call the main function after declaring it
main();
```

And the full `main` function will look like this:

```ts
async function main() {
  try {
    const assistant = await openai.beta.assistants.create({
      name: "Math Tutor",
      instructions:
        "You are a personal math tutor. Answer questions briefly, in a sentence or less.",
      tools: [
        { type: "code_interpreter" },
        {
          type: "function",
          function: quizJson,
        },
      ],
      // will work much better with the new model
      model: "gpt-4-1106-preview",
      // model: "gpt-3.5-turbo-1106",
    });

    // Log a first greeting
    console.log(
      "\nHello there, I'm Fernando's personal Math assistant. We'll start with a small quiz.\n",
    );

    // Create a thread
    const thread = await openai.beta.threads.create();

    // Use continueConversation as state for keep asking questions
    let continueConversation = true;

    while (continueConversation) {
      const userQuestion = isQuizAnswered
        ? await askRLineQuestion("You next question to the model: \n")
        // this will make the model  build a quiz using our provided function
        : "Make a quiz with 2 questions: One open ended, one multiple choice" +
          "Then, give me feedback for the responses.";

      // Pass in the user question into the existing thread
      await openai.beta.threads.messages.create(thread.id, {
        role: "user",
        content: userQuestion,
      });

      // Use runs to wait for the assistant response and then retrieve it
      const run = await openai.beta.threads.runs.create(thread.id, {
        assistant_id: assistant.id,
      });

      let actualRun = await openai.beta.threads.runs.retrieve(
        thread.id,
        run.id,
      );

      // Polling mechanism to see if actualRun is completed
      while (
        actualRun.status === "queued" ||
        actualRun.status === "in_progress" ||
        actualRun.status === "requires_action"
      ) {
        // requires_action means that the assistant is waiting for the functions to be added

        if (actualRun.status === "requires_action") {
          // extra single tool call
          const toolCall =
            actualRun.required_action?.submit_tool_outputs?.tool_calls[0];

          const name = toolCall?.function.name;

          const args = JSON.parse(toolCall?.function?.arguments || "{}");
          const questions = args.questions;

          const responses = await displayQuiz(name || "cool quiz", questions);

          // toggle flag that sets initial quiz
          isQuizAnswered = true;

          // we must submit the tool outputs to the run to continue
          await openai.beta.threads.runs.submitToolOutputs(
            thread.id,
            run.id,
            {
              tool_outputs: [
                {
                  tool_call_id: toolCall?.id,
                  output: JSON.stringify(responses),
                },
              ],
            },
          );
        }
        // keep polling until the run is completed
        await new Promise((resolve) => setTimeout(resolve, 2000));
        actualRun = await openai.beta.threads.runs.retrieve(thread.id, run.id);
      }

      // Get the last assistant message from the messages array
      const messages = await openai.beta.threads.messages.list(thread.id);

      // Find the last message for the current run
      const lastMessageForRun = messages.data
        .filter(
          (message) =>
            message.run_id === run.id && message.role === "assistant",
        )
        .pop();

      // If an assistant message is found, console.log() it
      if (lastMessageForRun) {
        // aparently the `content` array is not correctly typed
        // content returns an of objects do contain a text object
        const messageValue = lastMessageForRun.content[0] as {
          text: { value: string };
        };

        console.log(`${messageValue?.text?.value} \n`);
      }

      // Then ask if the user wants to ask another question and update continueConversation state
      const continueAsking = await askRLineQuestion(
        "Do you want to keep having a conversation? (yes/no) ",
      );

      continueConversation = continueAsking.toLowerCase() === "yes";

      // If the continueConversation state is falsy show an ending message
      if (!continueConversation) {
        console.log("Alrighty then, I hope you learned something!\n");
      }
    }

    // close the readline
    readline.close();
  } catch (error) {
    console.error(error);
  }
}
```

If you've created a new node project using `npm init` —recommended— you can add a script to run your project as follows

```json
{
  "scripts": {
    "start": "ts-node yourFileName.ts"
  }
}
```


