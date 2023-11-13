Just a week has passed since the OpenAI Dev Conf 2023, and a surprising revelation unfolded: The Assistant's API. Unveiled in the latest OpenAI blog [post](https://openai.com/blog/new-models-and-developer-products-announced-at-devday), the Assistants API marks a significant stride in empowering developers to craft agent-like experiences within their applications.

In their own words:

> Today, we’re releasing the Assistants API, our first step towards helping developers build agent-like experiences within their own applications. An assistant is a purpose-built AI that has specific instructions, leverages extra knowledge, and can call models and tools to perform tasks.

So, what does this mean for us? In essence, it means we now have the capability to construct our own AI assistants using the OpenAI API. This empowers us to harness the prowess of the GPT-4 model to execute tasks and answer questions with additional knowledge and context that we provide through uploaded documents (with no need for third-party involvement!), a Python code interpreter operating in a sandboxed environment, and a feature that truly caught my attention: function calling.

While functions themselves aren't novel, it's the way they've been implemented that truly stands out. In the past, calling a function meant uncertainty about the model's return, necessitating post-processing for the desired result — and even then, success wasn't guaranteed. Now, we can specify the desired function output, and the model will strive to provide a response aligning with the provided schema.

This newfound tool grants us immense flexibility and power, enabling our assistant to perform virtually any task — from sending emails to making calls and querying databases. The possibilities are limitless.

While the available documents and examples are still somewhat scarce, my curiosity led me to dive in and explore the potential. In this journey, I set out to build a simple math assistant, serving as a proof of concept for implementing function calls within the new Assistant API in Node.js.
