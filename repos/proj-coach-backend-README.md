## This is our main backend service. Written using NestJS

### Development setup

We use bun to run the project. to install
```bash
$ bun install
```

## Compile and run the project

```bash
# development
$ bun dev

# production mode
$ bun run build
$ bpm run start:prod
```

### Run tests

```bash
# unit tests
$ bun run test
```

### Potential issues when running bun install

#### 1. node-canvas errors: 
If you see errors like "Cannot find module '../build/Release/canvas.node'" when running the project. Canvas need to be rebuild for your system. Run `bun uninstall canvas && bun install canvas` to (try compile the binary)[https://github.com/Automattic/node-canvas?tab=readme-ov-file#compiling].


#### 2. 404 errors during `bun install`

If **you AREALDY exported `NPM_TOKEN` (and added it to your shell config)** but still see:

error: GET https://registry.npmjs.org/@dodo-world%2fproj-coach-schemas - 404
error: @dodo-world/proj-coach-schemas@0.11.5 failed to resolve


then Bun isn’t using it.

You may create a `bunfig.toml` in the project root:

```toml
[install.scopes]
"@dodo-world" = { token = "$NPM_TOKEN", url = "https://npm.pkg.github.com" }

```


## LangSmith Observability

The chatbot service is instrumented with [LangSmith](https://smith.langchain.com) for full observability of LLM calls, tool executions, and conversation flows.

### Setup

LangSmith is configured in `config/config.toml` under the `[langsmith]` section:

```toml
[langsmith]
tracing = true
endpoint = "https://api.smith.langchain.com"
apiKey = "<LANGSMITH_API_KEY>"
project = "Ditto-Chatbot"
projectId = "148a3494-b204-4447-901e-b01561c01a57"
```

The application will automatically read these settings and enable tracing. You can also override them via environment variables (`LANGSMITH_PROJECT`, `LANGSMITH_PROJECT_ID`, etc.) if needed.

### What's Traced

- **Graph Execution**: Full LangGraph workflow with chat ID and user ID metadata
- **LLM Calls**: All OpenAI model invocations with conversation context
- **Tool Executions**: Individual tool calls (pause/resume account, matchmaking updates, etc.)
- **Error Handling**: Tool failures and error responses

All traces are tagged with `chatbot`, `Ditto-Chatbot`, and individual `chat:<chatId>` tags for easy filtering in the LangSmith dashboard.

### Visualizing the Graph in LangSmith Studio

To view and debug your chatbot graph visually in **LangSmith Studio**:

1. **Ensure LangSmith is configured** in `config/config.toml`

2. **Export environment variables for the LangGraph CLI**:
```bash
export LANGSMITH_TRACING=true
export LANGSMITH_API_KEY=<LANGSMITH_API_KEY>
export LANGSMITH_PROJECT=Ditto-Chatbot
export OPENAI_API_KEY=<OPENAI_API_KEY>
```

3. **Start the LangGraph development server**:
```bash
bun run langgraph:dev
```

This will start the LangGraph API server at `http://localhost:8123` with hot-reloading.

4. **Open LangSmith Studio** in your browser at `http://localhost:8123/studio`

5. **Interact with your graph**:
   - View the chatbot graph structure visually
   - Run test invocations with sample inputs
   - Debug node execution and state transitions
   - Inspect tool calls and LLM responses in real-time

**Note**: The LangGraph server runs independently from your main NestJS application. It's purely for development and debugging purposes.

## Deployment

When you're ready to deploy your NestJS application to production, there are some key steps you can take to ensure it runs as efficiently as possible. Check out the [deployment documentation](https://docs.nestjs.com/deployment) for more information.

If you are looking for a cloud-based platform to deploy your NestJS application, check out [Mau](https://mau.nestjs.com), our official platform for deploying NestJS applications on AWS. Mau makes deployment straightforward and fast, requiring just a few simple steps:

```bash
$ npm install -g mau
$ mau deploy
```

With Mau, you can deploy your application in just a few clicks, allowing you to focus on building features rather than managing infrastructure.

## Resources

Check out a few resources that may come in handy when working with NestJS:

- Visit the [NestJS Documentation](https://docs.nestjs.com) to learn more about the framework.
- For questions and support, please visit our [Discord channel](https://discord.gg/G7Qnnhy).
- To dive deeper and get more hands-on experience, check out our official video [courses](https://courses.nestjs.com/).
- Deploy your application to AWS with the help of [NestJS Mau](https://mau.nestjs.com) in just a few clicks.
- Visualize your application graph and interact with the NestJS application in real-time using [NestJS Devtools](https://devtools.nestjs.com).
- Need help with your project (part-time to full-time)? Check out our official [enterprise support](https://enterprise.nestjs.com).
- To stay in the loop and get updates, follow us on [X](https://x.com/nestframework) and [LinkedIn](https://linkedin.com/company/nestjs).
- Looking for a job, or have a job to offer? Check out our official [Jobs board](https://jobs.nestjs.com).

## Support

Nest is an MIT-licensed open source project. It can grow thanks to the sponsors and support by the amazing backers. If you'd like to join them, please [read more here](https://docs.nestjs.com/support).

## Stay in touch

- Author - [Kamil Myśliwiec](https://twitter.com/kammysliwiec)
- Website - [https://nestjs.com](https://nestjs.com/)
- Twitter - [@nestframework](https://twitter.com/nestframework)

## License

Nest is [MIT licensed](https://github.com/nestjs/nest/blob/master/LICENSE).
