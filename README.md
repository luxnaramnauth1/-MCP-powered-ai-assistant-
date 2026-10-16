# MCP Banking Assistant

An AI assistant that answers banking questions by calling tools on a **Model Context Protocol (MCP) server**. The server holds the data and the rules for access. The assistant can be a Claude model that chooses the tools, or a rule-based router that runs with no API key.

> The data is **synthetic**. No real customer data is included.

## Why this project matters

Language models are good at reasoning and poor at knowing facts about a bank's data. MCP gives a model a standard, safe way to ask for facts. This project shows the full loop: a typed server, a client that uses it, privacy controls on what leaves the server, and checks on what the model says back.

## Architecture

```
            question
               |
               v
  +------------------------+   tool calls    +-------------------------+
  |  Assistant (client)    | --------------> |  Bank MCP server        |
  |  claude mode: Claude   |  <------------- |  4 read-only tools      |
  |  offline mode: router  |   JSON results  |  masks customer IDs     |
  +------------------------+                 +-------------------------+
               |                                        |
               v                                        v
     guardrail: every number            cleaned, masked data
     must come from a tool result       (CSV loaded once at start)
               |
               v
            answer
```

## Files

| File | Purpose |
|---|---|
| `server/bank_mcp_server.py` | MCP server with four tools: `list_accounts`, `account_summary`, `search_transactions`, `flag_high_value_debits` |
| `assistant/assistant.py` | MCP client, Claude tool-use loop, offline router, guardrail |
| `assistant/chat.py` | Command-line chat using the assistant |
| `banking_mcp_assistant.ipynb` | Notebook: starts the server, calls tools, asks questions, runs the privacy and guardrail checks, runs the tests. Saved with outputs |
| `data/bank_transactions.csv` | 223 synthetic transactions for six accounts, with data quality problems |
| `requirements.txt` | Pinned versions |

## Problems and solutions

| # | Problem | Why it matters | Solution in this project | Where to see it |
|---|---|---|---|---|
| 1 | A model could see raw customer IDs | Personal data must not reach an AI or its logs | Customer IDs are replaced by salted SHA-256 tokens when the data loads, and the raw column is dropped. Tools only return tokens | `load_clean()` in the server; privacy check in notebook section 5 |
| 2 | A model could invent a number | A made-up balance or total in a bank answer is harmful | Guardrail: every number in an answer must appear in a tool result or the question. Invented numbers are rejected | `passes_guardrail()`; notebook sections 4 and 7 |
| 3 | The data is messy (text numbers, stray spaces, impossible dates, blanks, exact copies) | Totals and comparisons go wrong silently | Cleaning at load time: types converted, text normalised, unusable rows set aside, exact copies removed. The counts are returned in every `list_accounts` call | `load_clean()`; `data_quality` field |
| 4 | Currencies would be added together | ACC2006 is in USD, the rest in MUR | Totals stay in each account's own currency. USD is converted to MUR only for threshold checks | `account_summary` and `flag_high_value_debits` |
| 5 | Tools can return huge results | Large outputs are slow and can leak more data than needed | Row limit of 50 on every search, and a `limit` parameter | `search_transactions` |
| 6 | Vague tools make model choices unreliable | The model needs a clear purpose for each tool | Each tool has typed parameters and a docstring that becomes its description | Tool list in notebook section 1 |
| 7 | A tool loop could run forever | Costs and waits grow without limit | `MAX_STEPS = 6` tool rounds per question | `_ask_claude()` |
| 8 | No API key or no internet | The project must still run for anyone | Offline mode: a rule-based router chooses the same tools and writes the answer from the result | `route_offline()`, `answer_from()` |
| 9 | Connections break when a notebook runs cell by cell | The server process could be left running or closed by mistake | The connection is held by one background task. Start and stop work from any cell | `start()`, `_hold_connection()`, `stop()` |
| 10 | Tests of the Claude path need a paid key | Results cannot be checked without cost | A scripted fake model follows the same tool-use protocol and checks the loop and the guardrail | notebook section 6 |
| 11 | Library changes can break MCP code | MCP's API is still changing | Pinned `mcp==1.27.2` and `anthropic==1.15.0` | `requirements.txt` |
| 12 | Model output could be used as if it were certain | Users need to know where a figure came from | The chat and the notebook show which tools produced each answer | `chat.py` output; notebook section 3 |

## How to run

1. Install the packages: `pip install -r requirements.txt`
2. Open `banking_mcp_assistant.ipynb` in Jupyter, VS Code or Colab, and run all cells. No API key is needed.
3. Optional, for the live Claude mode: `export ANTHROPIC_API_KEY=...` before starting Jupyter, then run the notebook again. Section 8 then runs a real question.
4. Or chat from the terminal: `python assistant/chat.py`. Example questions:
   - `summary for ACC2003`
   - `show large debits over 10000`
   - `failed transactions on ACC2002`
   - `what accounts do you have?`

## Test results

The notebook runs 15 checks, and all of them pass:

- the server exposes exactly four tools
- an account's net flow equals credits minus debits
- large-debit results are at or above the threshold and sorted largest first
- search honours the row limit
- an unknown account returns an error with the valid account list
- the router sends totals and large-debit questions to the right tools
- the guardrail accepts a correct answer and rejects an invented number
- no raw customer ID appears in any tool output
- the Claude tool-use loop calls the tool and passes the guardrail (scripted model)

## Privacy notes

- Masking is not anonymisation. Anyone who knows the salt and a candidate customer ID can check the token. Set your own salt with the `TOKEN_SALT` environment variable and keep it secret.
- The server is read-only. No tool writes to the data.
- The data is held in memory while the server runs.

## Limits

- The live Claude path has not been run against the API, because no key was available. Its loop and guardrail are tested with a scripted model.
- The offline router understands a fixed set of question patterns. Claude mode handles more varied wording.
- The exchange rate of 42 MUR per USD is a fixed demo value.
- The server uses stdio only, with one client. A production server would add authentication, logging of each call, and an HTTP transport.
- The data is loaded once at start-up. A production system would read from a database and refresh it.

## Going further

- Add a tool that reads from a real database and check each query for SQL injection
- Add a Streamlit or Gradio front end over the same assistant
- Log every tool call with a timestamp for audit
- Run the server in Docker, and add a test job in GitHub Actions
