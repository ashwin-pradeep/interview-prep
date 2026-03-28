# AI in Development

> **Target Audience:** Senior Front End Developer Interview Preparation

## Table of Contents

1. [AI Fundamentals for Developers](#ai-fundamentals-for-developers)
2. [AI-Powered Development Tools](#ai-powered-development-tools)
3. [AI Code Assistants](#ai-code-assistants)
4. [Integrating AI into Web Applications](#integrating-ai-into-web-applications)
5. [Prompt Engineering](#prompt-engineering)
6. [AI-Powered UI/UX](#ai-powered-uiux)
7. [Testing & Quality with AI](#testing--quality-with-ai)
8. [Ethics, Risks & Best Practices](#ethics-risks--best-practices)
9. [AI Architecture & Performance](#ai-architecture--performance)
10. [Future of AI in Frontend](#future-of-ai-in-frontend)

---

## AI Fundamentals for Developers

### What AI concepts should frontend developers understand?

| Concept | Description | Frontend Relevance |
|---|---|---|
| **LLM** | Large Language Model | Chatbots, code generation, content |
| **Token** | Text unit processed by LLM | API cost, context limits |
| **Prompt** | Input text to the AI model | UX design for AI features |
| **Inference** | Model generates output | Latency, streaming UX |
| **Fine-tuning** | Customizing a model | Domain-specific features |
| **Embedding** | Vector representation of text | Search, recommendations |
| **RAG** | Retrieval-Augmented Generation | Grounded AI responses |
| **Hallucination** | AI generates false information | Validation, disclaimers |

---

### What is the difference between AI, ML, and LLMs?

```
Artificial Intelligence (AI)
  └── Machine Learning (ML)
        ├── Supervised Learning (classification, regression)
        ├── Unsupervised Learning (clustering, dimensionality reduction)
        └── Deep Learning
              └── Large Language Models (GPT, Claude, Gemini)
```

| Term | Scope | Example |
|---|---|---|
| **AI** | Broad — machines mimicking intelligence | Siri, self-driving cars |
| **ML** | Learning from data | Spam filter, recommendation engine |
| **Deep Learning** | Neural networks with many layers | Image recognition |
| **LLM** | Language-specific deep learning | ChatGPT, Claude, Copilot |

---

### What are tokens and context windows?

**Tokens** are the basic units LLMs process (roughly 4 characters or ~¾ of a word in English).

```
"Hello, world!" → ["Hello", ",", " world", "!"] → 4 tokens
```

| Model | Context Window | Approx. Pages |
|---|---|---|
| GPT-3.5 | 4K–16K tokens | 3–12 pages |
| GPT-4 | 8K–128K tokens | 6–100 pages |
| Claude 3 | 200K tokens | ~150 pages |
| Gemini 1.5 | 1M+ tokens | ~750 pages |

**Frontend impact:** Context window limits how much code/docs you can send in one request. Design chunking strategies for large codebases.

---

## AI-Powered Development Tools

### What AI coding tools are available for developers?

| Tool | Category | Key Feature |
|---|---|---|
| **GitHub Copilot** | Code completion | Inline suggestions in IDE |
| **Cursor** | AI-first IDE | Full codebase awareness |
| **ChatGPT / Claude** | General AI | Code generation, debugging, explanations |
| **Codeium** | Code completion | Free alternative to Copilot |
| **Amazon CodeWhisperer** | Code completion | AWS integration |
| **Tabnine** | Code completion | On-premise / privacy-focused |
| **v0 by Vercel** | UI generation | React component generation |
| **Bolt / Lovable** | Full-stack generation | App scaffolding from prompts |

---

### How does GitHub Copilot work under the hood?

```
Developer types code → IDE sends context to Copilot API
  → Model (Codex / GPT-4) processes context
  → Returns completion suggestions
  → IDE displays inline (ghost text)
  → Developer accepts/rejects
```

**Context sent to Copilot:**
- Current file content
- Open files in IDE
- File path and language
- Cursor position
- Comments and docstrings

**Best practices for better suggestions:**
1. Write clear comments describing intent
2. Use descriptive function/variable names
3. Provide type annotations
4. Keep related code in nearby files
5. Write a few examples before asking for patterns

---

### How do AI tools help with code review?

| Tool | Capability |
|---|---|
| **GitHub Copilot for PRs** | Auto-generates PR descriptions, reviews code |
| **CodeRabbit** | AI-powered code review comments |
| **Sourcery** | Refactoring suggestions |
| **SonarQube AI** | Code quality + AI insights |

```yaml
# Example: AI-powered PR review in GitHub Actions
name: AI Code Review
on: [pull_request]
jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: coderabbitai/ai-pr-reviewer@latest
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
```

---

## AI Code Assistants

### How do you effectively use AI for code generation?

**Effective patterns:**

```
1. Describe the WHAT and WHY, not just the HOW:
   ❌ "Write a function"
   ✅ "Write a debounce function that delays execution until the user
       stops typing for 300ms, cancels pending calls, and supports
       TypeScript generics"

2. Provide context:
   ❌ "Sort this array"
   ✅ "Sort this array of user objects by lastName, then firstName,
       case-insensitive, using Intl.Collator for locale-aware sorting"

3. Specify constraints:
   "Generate a React hook for infinite scrolling that:
    - Uses Intersection Observer
    - Supports both window and container scrolling
    - Returns loading/error/hasMore states
    - Is compatible with React Query's useInfiniteQuery
    - Includes TypeScript types"
```

---

### How do you use AI for debugging?

```
Effective debugging prompt structure:
1. Describe the expected behavior
2. Describe the actual behavior
3. Provide the relevant code
4. Include error messages / stack traces
5. List what you've already tried

Example:
"My React component re-renders infinitely when I add a useEffect
 dependency. Expected: fetch data once on mount. Actual: infinite
 API calls.

 [paste code]

 Error in console: 'Maximum update depth exceeded'

 I tried adding an empty dependency array but then the data
 is stale when props change."
```

---

### What are the limitations of AI code generation?

| Limitation | Impact | Mitigation |
|---|---|---|
| **Hallucination** | Non-existent APIs, wrong syntax | Always verify, test generated code |
| **Stale knowledge** | Outdated library versions | Specify versions, check docs |
| **Security** | May suggest vulnerable patterns | Review for security, run SAST tools |
| **Copyright** | Training data concerns | Review licenses, use allowlists |
| **Context limits** | Can't see full codebase | Provide key files, use RAG |
| **Over-reliance** | Skills atrophy | Understand before accepting |

---

## Integrating AI into Web Applications

### How do you integrate an LLM API into a frontend app?

```jsx
// Streaming chat with OpenAI API (via backend proxy)
function useChat() {
  const [messages, setMessages] = useState([]);
  const [isStreaming, setIsStreaming] = useState(false);

  const sendMessage = async (content) => {
    const userMessage = { role: "user", content };
    const updatedMessages = [...messages, userMessage];
    setMessages(updatedMessages);
    setIsStreaming(true);

    try {
      const response = await fetch("/api/chat", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ messages: updatedMessages }),
      });

      const reader = response.body.getReader();
      const decoder = new TextDecoder();
      let assistantMessage = "";

      setMessages((prev) => [...prev, { role: "assistant", content: "" }]);

      while (true) {
        const { done, value } = await reader.read();
        if (done) break;

        assistantMessage += decoder.decode(value);
        setMessages((prev) => {
          const updated = [...prev];
          updated[updated.length - 1] = {
            role: "assistant",
            content: assistantMessage,
          };
          return updated;
        });
      }
    } finally {
      setIsStreaming(false);
    }
  };

  return { messages, sendMessage, isStreaming };
}
```

---

### How do you build a streaming AI chat UI?

```jsx
function ChatInterface() {
  const { messages, sendMessage, isStreaming } = useChat();
  const [input, setInput] = useState("");
  const bottomRef = useRef(null);

  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [messages]);

  const handleSubmit = (e) => {
    e.preventDefault();
    if (!input.trim() || isStreaming) return;
    sendMessage(input);
    setInput("");
  };

  return (
    <div className="chat-container">
      <div className="messages">
        {messages.map((msg, i) => (
          <div key={i} className={`message ${msg.role}`}>
            <ReactMarkdown>{msg.content}</ReactMarkdown>
          </div>
        ))}
        {isStreaming && <TypingIndicator />}
        <div ref={bottomRef} />
      </div>
      <form onSubmit={handleSubmit}>
        <input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          placeholder="Ask something..."
          disabled={isStreaming}
        />
        <button type="submit" disabled={isStreaming || !input.trim()}>
          Send
        </button>
      </form>
    </div>
  );
}
```

---

### What is RAG (Retrieval-Augmented Generation)?

RAG enhances AI responses by retrieving relevant documents before generation.

```
User Query → Embed Query → Search Vector DB → Retrieve Relevant Docs
  → Combine (Query + Docs) → Send to LLM → Generate Grounded Response
```

```js
// Simplified RAG pipeline
async function ragQuery(userQuestion) {
  // 1. Embed the question
  const embedding = await embedText(userQuestion);

  // 2. Search vector database for relevant docs
  const relevantDocs = await vectorDB.search(embedding, { topK: 5 });

  // 3. Build context-rich prompt
  const context = relevantDocs.map((d) => d.content).join("\n\n");
  const prompt = `
    Answer based on the following context. If the answer isn't in the context, say so.

    Context:
    ${context}

    Question: ${userQuestion}
  `;

  // 4. Send to LLM
  return await llm.generate(prompt);
}
```

**Use cases:** Documentation chatbots, customer support, knowledge base search, codebase Q&A.

---

## Prompt Engineering

### What are key prompt engineering techniques?

| Technique | Description | Example |
|---|---|---|
| **Zero-shot** | No examples provided | "Translate this to French" |
| **Few-shot** | Include examples | "Translate: cat→chat, dog→chien, bird→?" |
| **Chain of Thought** | Step-by-step reasoning | "Think step by step..." |
| **Role prompting** | Assign a persona | "You are a senior React developer..." |
| **System prompt** | Set behavior rules | "Always respond in JSON format" |
| **Constrained output** | Limit format | "Respond with only yes or no" |

```js
// System prompt for a code review bot
const systemPrompt = `You are a senior frontend code reviewer.
Review the code for:
1. Performance issues
2. Security vulnerabilities
3. Accessibility problems
4. React best practices

Format your response as:
- 🔴 Critical: [issue]
- 🟡 Warning: [issue]
- 🟢 Suggestion: [improvement]

Be concise and actionable.`;
```

---

### How do you design prompts for consistent code generation?

```
Template approach:
1. Role: "You are a TypeScript React developer using modern hooks"
2. Context: "Working on a Next.js 14 App Router project with Tailwind CSS"
3. Task: "Create a paginated data table component"
4. Constraints:
   - Use TypeScript with strict types
   - Follow the existing component pattern: [paste example]
   - Use server components where possible
   - Include error and loading states
   - Add JSDoc comments
5. Output format: "Return only the code, no explanations"
```

---

## AI-Powered UI/UX

### How can AI enhance user interfaces?

| Feature | Implementation | Example |
|---|---|---|
| **Smart search** | Semantic search with embeddings | "Find that email about the budget" |
| **Autocomplete** | Context-aware suggestions | Gmail Smart Compose |
| **Personalization** | Recommend based on behavior | Netflix, Spotify |
| **Content generation** | Generate text, images | Notion AI, Canva Magic Write |
| **Accessibility** | Alt text generation, captions | Automatic image descriptions |
| **Chatbots** | Conversational UI | Customer support, onboarding |

```jsx
// AI-powered search with semantic matching
function SmartSearch() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState([]);

  const search = useDebouncedCallback(async (q) => {
    if (!q.trim()) return setResults([]);
    const response = await fetch(`/api/semantic-search?q=${encodeURIComponent(q)}`);
    setResults(await response.json());
  }, 300);

  return (
    <div>
      <input
        value={query}
        onChange={(e) => { setQuery(e.target.value); search(e.target.value); }}
        placeholder="Search naturally..."
      />
      {results.map((result) => (
        <SearchResult key={result.id} {...result} score={result.similarity} />
      ))}
    </div>
  );
}
```

---

### How do you handle AI loading states and UX patterns?

```jsx
// Skeleton + streaming pattern
function AIResponse({ isLoading, isStreaming, content, error }) {
  if (error) {
    return (
      <div className="error">
        <p>Something went wrong. Please try again.</p>
        <button onClick={retry}>Retry</button>
      </div>
    );
  }

  if (isLoading) {
    return (
      <div className="ai-loading">
        <Skeleton lines={3} />
        <span className="thinking">AI is thinking...</span>
      </div>
    );
  }

  return (
    <div className={`ai-response ${isStreaming ? "streaming" : ""}`}>
      <ReactMarkdown>{content}</ReactMarkdown>
      {isStreaming && <CursorBlink />}
    </div>
  );
}
```

**UX best practices for AI features:**
- Show progress/thinking indicators
- Stream responses token by token
- Allow cancellation of long requests
- Show confidence scores when applicable
- Provide feedback mechanisms (thumbs up/down)
- Add disclaimers for AI-generated content

---

## Testing & Quality with AI

### How do you use AI for test generation?

```
Prompt: "Generate unit tests for this function using Jest and
React Testing Library. Cover: happy path, edge cases, error
states, accessibility, and user interactions.

Function:
[paste component code]

Follow existing test patterns:
[paste example test]"
```

```jsx
// AI-generated test (always review!)
describe("SearchInput", () => {
  it("debounces search by 300ms", async () => {
    jest.useFakeTimers();
    const onSearch = jest.fn();
    render(<SearchInput onSearch={onSearch} />);

    const input = screen.getByRole("searchbox");
    await userEvent.type(input, "react");

    expect(onSearch).not.toHaveBeenCalled();
    jest.advanceTimersByTime(300);
    expect(onSearch).toHaveBeenCalledWith("react");
  });

  it("cancels pending search on new input", async () => {
    jest.useFakeTimers();
    const onSearch = jest.fn();
    render(<SearchInput onSearch={onSearch} />);

    const input = screen.getByRole("searchbox");
    await userEvent.type(input, "re");
    jest.advanceTimersByTime(200);
    await userEvent.type(input, "act");
    jest.advanceTimersByTime(300);

    expect(onSearch).toHaveBeenCalledTimes(1);
    expect(onSearch).toHaveBeenCalledWith("react");
  });
});
```

---

### How do you validate AI-generated code?

| Check | Method |
|---|---|
| **Correctness** | Run tests, manual review |
| **Security** | SAST tools (Snyk, SonarQube), code review |
| **Performance** | Lighthouse, bundle analysis |
| **Types** | TypeScript strict mode |
| **Style** | ESLint, Prettier |
| **Accessibility** | axe-core, keyboard testing |
| **License** | Check for copyrighted code patterns |

---

## Ethics, Risks & Best Practices

### What are the risks of using AI in development?

| Risk | Description | Mitigation |
|---|---|---|
| **Security vulnerabilities** | AI may suggest insecure code | Security reviews, SAST |
| **License issues** | Training data may include licensed code | Code origin review |
| **Over-reliance** | Developer skills atrophy | Understand before accepting |
| **Bias** | Models reflect training data biases | Test with diverse inputs |
| **Data privacy** | Sending code to external APIs | Use enterprise/on-premise options |
| **Hallucination** | Confident but wrong answers | Always verify |
| **Cost** | API usage can be expensive | Rate limiting, caching |

---

### What are best practices for AI-assisted development?

1. **Review everything** — treat AI output as a first draft
2. **Test AI code** — write tests before or after generation
3. **Understand the code** — don't merge what you can't explain
4. **Use for acceleration, not replacement** — augment your skills
5. **Protect sensitive data** — don't send secrets to AI APIs
6. **Version prompts** — track and iterate on system prompts
7. **Set guardrails** — content filters, output validation
8. **Monitor costs** — track token usage and API costs
9. **Stay updated** — AI tools evolve rapidly
10. **Document AI usage** — note when AI generates significant code

---

## AI Architecture & Performance

### How do you optimize AI API calls in a frontend app?

```js
// 1. Caching responses
const responseCache = new Map();

async function cachedAIQuery(prompt) {
  const cacheKey = hashPrompt(prompt);
  if (responseCache.has(cacheKey)) return responseCache.get(cacheKey);
  const response = await fetchAI(prompt);
  responseCache.set(cacheKey, response);
  return response;
}

// 2. Debounce user input before sending to AI
const debouncedSuggest = debounce(async (input) => {
  const suggestion = await fetchAI(`Complete: ${input}`);
  setSuggestion(suggestion);
}, 500);

// 3. Use streaming for long responses
async function streamResponse(prompt, onChunk) {
  const response = await fetch("/api/ai/stream", {
    method: "POST",
    body: JSON.stringify({ prompt }),
  });

  const reader = response.body.getReader();
  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    onChunk(new TextDecoder().decode(value));
  }
}

// 4. Abort on component unmount or new request
function useAIQuery(prompt) {
  const [result, setResult] = useState(null);

  useEffect(() => {
    const controller = new AbortController();
    fetchAI(prompt, { signal: controller.signal })
      .then(setResult)
      .catch((e) => {
        if (e.name !== "AbortError") console.error(e);
      });
    return () => controller.abort();
  }, [prompt]);

  return result;
}
```

---

## Future of AI in Frontend

### What AI trends should frontend developers watch?

| Trend | Description | Impact |
|---|---|---|
| **AI-native frameworks** | Frameworks designed around AI (Vercel AI SDK) | New development patterns |
| **Local/Edge AI** | Models running in browser (WebGPU, ONNX.js) | Offline AI features, privacy |
| **Multi-modal AI** | Text + image + audio understanding | Richer UI interactions |
| **AI agents** | Autonomous task completion | Automated coding workflows |
| **Code generation** | Full app generation from descriptions | Rapid prototyping |
| **AI design tools** | Figma AI, design-to-code | Faster design-to-dev pipeline |

```js
// Browser-based AI with WebLLM (emerging)
import { CreateMLCEngine } from "@mlc-ai/web-llm";

const engine = await CreateMLCEngine("Llama-3-8B-Instruct-q4f16_1", {
  initProgressCallback: (progress) => {
    console.log(`Loading model: ${(progress.progress * 100).toFixed(1)}%`);
  },
});

const response = await engine.chat.completions.create({
  messages: [{ role: "user", content: "Explain React hooks" }],
  stream: true,
});
```

---

[← Back to README](./README.md)
