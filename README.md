# SQLite Field Notes

SQLite Field Notes is a documentation assistant that answers questions about SQLite and shows the evidence behind its answers. It searches a small collection of SQLite documentation summaries, sends the most relevant passages to Groq, and links each cited passage to the official docs. When the available evidence is insufficient, the assistant says so.

**[Try the live demo](https://sqlite-docs-assistant.vercel.app)**

| Try asking | What to look for |
| --- | --- |
| Can readers and writers run together in WAL mode? | Readers and a writer can run concurrently. The source card links to the write-ahead logging docs. |
| How do I enable foreign key enforcement? | The answer mentions `PRAGMA foreign_keys = ON` and explains that it applies per connection. |
| How much does SQLite cloud hosting cost? | The assistant declines because its indexed material contains no pricing evidence. |

## Run locally

Requires **Node.js 24** and npm. The project uses Node's built-in HTTP server and fetch API, so there are no packages to install. Run the commands from the project root.

1. For generated answers, copy `.env.example` to `.env` and set the variable below. Keep `.env` out of version control. [Create a Groq API key](https://console.groq.com/keys).

   ```dotenv
   GROQ_API_KEY=your_groq_api_key
   ```

2. Start the app.

   ```sh
   npm start
   ```

3. Open [localhost:3000](http://127.0.0.1:3000). With no API key, supported queries still show relevant passages in retrieval-only mode.

## How it works

1. The browser sends a question to `POST /api/ask`. The server accepts trimmed questions of up to 500 characters.
2. BM25 ranks eight curated SQLite documentation summaries. The server selects up to three passages with positive scores. If there are no matches, it returns an insufficient-evidence response without calling Groq.
3. Groq's `openai/gpt-oss-20b` receives the question and retrieved passages and returns an answer and citation IDs in a strict JSON schema. [Groq model documentation](https://console.groq.com/docs/model/openai/gpt-oss-20b)
4. The server checks that the cited IDs belong to the retrieved set. It declines if the model returns an empty answer or invalid citations. The UI renders accepted citations with the indexed summary and a link to the official page.

The collection covers transactions, write-ahead logging, foreign keys, indexes, UPSERT, PRAGMA settings, in-memory databases, and backups. The source cards contain curated summaries of those pages. Citation validation checks source identity, while judging whether a passage supports every claim still requires reading it.

## Evaluation

```sh
npm run eval
```

The evaluation uses 20 hand-written questions. Eighteen have a labeled source, and two are out of scope. Top-k measures whether the expected source appears within the first k results.

| Retrieval method | Top-1 hits | Top-3 hits | Out-of-scope questions with no match |
| --- | ---: | ---: | ---: |
| Word overlap | 18/18 | 18/18 | 2/2 |
| BM25 | 18/18 | 18/18 | 2/2 |

Both methods tied on this small, topic-aligned set. Paraphrases with little vocabulary overlap and out-of-scope questions that mention an indexed topic would make a harder follow-up set. Mocked Groq checks cover retrieval-only mode, accepted citations, rejected citation IDs, and skipping the provider when retrieval is empty. These results measure source retrieval and response handling. Live answer quality and semantic retrieval are outside the current evaluation.

## Deploy to Vercel

1. Import the repository into Vercel. The app serves `public/index.html` and runs `api/ask.js` as a server-side function.
2. Add `GROQ_API_KEY` as a **sensitive** variable for **Production** in the Vercel project settings.
3. Deploy or redeploy, then run one supported question and one out-of-scope question from the examples above. [Vercel environment variables](https://vercel.com/docs/environment-variables)

The public answer endpoint uses the project's Groq quota. Apply a request rate limit to `/api/ask` before sharing a deployment backed by your key.

## Project layout

| Path | Responsibility |
| --- | --- |
| `public/index.html` | Question form, examples, and source cards |
| `api/ask.js` | Request validation, Groq call, citation checks, and fallback responses |
| `lib/docs.js` | Curated passages and official SQLite links |
| `lib/search.js` | BM25 retrieval and the word-overlap baseline |
| `eval.js` | Labeled retrieval cases and mocked provider checks |
| `server.js` | Local HTTP entry point |
| `vercel.json` | Hosting configuration |
