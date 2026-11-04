# FutureWise

**An LLM-based platform for explainable financial news analysis and risk-aware portfolio intelligence for Indian retail investors.**

FutureWise takes a user's stock portfolio, pulls the latest news for each holding from trusted Indian RSS feeds, and uses a quantised, finance-tuned Llama-2 model to turn that news into a structured, plain-language report. No finance background is needed to read it.

> B.Tech (CSE-AIML) Semester VII project, D.Y. Patil College of Engineering & Technology, Kolhapur, 2026-27.

---

## Why FutureWise?

Retail investors in India often hold many stocks but don't have time to read the news for each one, and plain news aggregators leave the interpretation to the user. FutureWise reads the news for you and explains what it means for each holding.

For every stock in your portfolio you get four sections:

| Section | What it tells you |
| --- | --- |
| **Positive Developments** | News that is favourable for the stock |
| **Potential Concerns** | Risks and negative signals |
| **Analysis** | A short reasoned explanation tying the news together |
| **Overall Impact** | `Positive`, `Negative` or `Neutral` |

Each analysis is shown with clickable source headlines, so every claim can be traced back to the news it came from.

## Features

- **Portfolio management**: add stocks with symbol, company name and quantity.
- **Live news ingestion**: top 5 recent, unique articles per stock from Google News India, The Economic Times and Moneycontrol (last 7 days only).
- **Explainable, structured output**: four fixed sections per stock, returned as JSON.
- **Low-hallucination generation**: strict zero-shot prompt, greedy decoding and a 350-token cap.
- **Runs on free hardware**: the 7B model is 4-bit quantised and fits on a single free-tier Google Colab Tesla T4 (15 GB VRAM).
- **Analysis history**: results are stored in MongoDB with timestamps, so revisiting a portfolio does not re-run the GPU.
- **Resilient networking**: long request timeouts (up to 10 minutes) in the Node.js backend to handle slow multi-stock inference.

## Architecture

FutureWise is a three-tier microservice system. The tiers talk to each other over HTTP/JSON.

```
┌──────────────────┐   HTTP/JSON   ┌────────────────────────┐   HTTPS (pyngrok)   ┌─────────────────────────────┐
│  React.js client │ ────────────► │  Node.js + Express API │ ──────────────────► │  Flask AI microservice      │
│  (portfolio UI)  │ ◄──────────── │  + MongoDB (Mongoose)  │ ◄────────────────── │  (Google Colab, Tesla T4)   │
└──────────────────┘               └────────────────────────┘                     │  RSS fetch → prompt → LLM   │
                                                                                  └─────────────────────────────┘
```

1. **Client layer (React.js)**: collects the portfolio, shows loading state during long inference, and renders the categorised results.
2. **Application & database layer (Node.js, Express, MongoDB)**: validates input, stores portfolios (`Portfolio` schema) and AI results (`Analysis` schema), and calls the AI service.
3. **AI inference layer (Flask on Google Colab)**: fetches and filters news, builds the prompt, runs the quantised model and returns a JSON array. It is exposed to the backend through a `pyngrok` HTTPS tunnel.

### Pipeline per stock

1. Build a company-specific search query and fetch RSS entries with `feedparser`.
2. Drop articles older than 7 days, de-duplicate headlines (case-insensitive), keep the 5 most recent.
3. Inject them into a strict `[INST]` zero-shot prompt that tells the model to act as a financial analyst and use only the supplied news.
4. Generate with greedy decoding (`do_sample=False`, `max_new_tokens=350`).
5. Parse into the four sections, return JSON, archive in MongoDB, and render in the UI.

## Tech Stack

| Layer | Technology |
| --- | --- |
| Frontend | React.js, Axios |
| Backend | Node.js, Express.js |
| Database | MongoDB with Mongoose |
| AI service | Python, Flask, flask-cors |
| News retrieval | `feedparser` over RSS (Google News India, Economic Times, Moneycontrol) |
| Base model | `meta-llama/Llama-2-7b-chat-hf` |
| Domain adapter | `FinGPT/fingpt-forecaster_dow30_llama2-7b_lora` (PEFT LoRA) |
| Quantisation | `bitsandbytes` 4-bit NF4, double quantisation, fp16 compute |
| Libraries | PyTorch, Transformers, PEFT, Accelerate |
| Tunnelling | `pyngrok` |
| Runtime | Google Colab, NVIDIA Tesla T4 (15 GB VRAM) |

## Getting Started

<!-- Author: check folder names, script names and environment variable names below against the repo and adjust. -->

### Prerequisites

- Node.js and npm
- A running MongoDB instance (local or Atlas)
- A Google account for Google Colab (free tier with a T4 GPU is enough)
- A Hugging Face account with **approved access to `meta-llama/Llama-2-7b-chat-hf`** (the model is gated)
- An ngrok account and auth token for the `pyngrok` tunnel

### 1. Clone the repository

```bash
git clone https://github.com/ashwinmali7781/FutureWise-Platform-for-investment-main.git
cd FutureWise-Platform-for-investment-main
```

### 2. Start the AI inference service (Google Colab)

1. Open the Flask/inference notebook in Google Colab and switch the runtime to **GPU (T4)**.
2. Add your Hugging Face token and ngrok auth token when prompted.
3. Run all cells. The notebook loads the 4-bit quantised Llama-2 model, attaches the FinGPT LoRA adapter, starts the Flask server in a background thread and prints a public `https://...ngrok...` URL.
4. Copy that URL. You will give it to the backend in the next step.

> The ngrok URL changes whenever the Colab session restarts, so update the backend configuration each time.

### 3. Start the backend

```bash
cd backend            # adjust to the actual folder name
npm install
```

Create a `.env` file with your MongoDB connection string and the Colab tunnel URL:

```env
MONGO_URI=<your MongoDB connection string>
AI_SERVICE_URL=<your ngrok URL from Colab>
PORT=5000
```

```bash
npm start
```

### 4. Start the frontend

```bash
cd frontend           # adjust to the actual folder name
npm install
npm start
```

Open the app in your browser, add a few stocks (for example `RELIANCE`, `TCS`, `INFY`) and run an analysis.

## Results

Measured on a single Google Colab Tesla T4 (15 GB VRAM):

| Metric | fp16 baseline | 4-bit NF4 (FutureWise) |
| --- | --- | --- |
| Model weights VRAM | ~13.5 GB | ~4.2 GB |
| Total peak VRAM | ~16.5 GB | ~7.2 GB |
| Execution status | Out-of-memory failure | Stable |
| Average latency per stock | N/A | ~45 to 60 s |
| Output length cap | N/A | 350 tokens |

- 4-bit quantisation cuts weight memory by roughly **68%**, which is what makes a 7B model usable on free-tier hardware. 8-bit was tried and was unstable on multi-stock queries.
- End-to-end latency grows roughly linearly with portfolio size, so the backend handles requests asynchronously with long timeouts.
- With the strict prompt and greedy decoding, every response followed the four-section structure.

## Limitations

- **Latency**: about 45 to 60 seconds per stock on a T4, so large portfolios take minutes. Cached analyses in MongoDB avoid repeat cost.
- **Colab dependence**: the AI service runs in a Colab session, so it needs to be restarted and re-tunnelled when the session ends.
- **Adapter domain**: the FinGPT Forecaster adapter was trained on US Dow 30 stocks. FutureWise grounds it with Indian RSS news, but it was not fine-tuned on NSE/BSE data.
- **News coverage**: analysis quality depends on what the free RSS feeds return for a given company.

## Future Scope

- Fine-tune on Indian market data (NSE/BSE).
- Move inference off Colab to a persistent GPU host to remove the tunnel and session limits.
- Batch or parallelise multi-stock inference to reduce wait time.

## Team

Department of Computer Science and Engineering (AI & ML), D.Y. Patil College of Engineering & Technology, Kolhapur.

| Name | Roll No. |
| --- | --- |
| Ashwin Mali | A15 |
| Sarthak Shinde | A17 |
| Pratik Gharge | A48 |
| Yash Shelar | A42 |
| Harshwardhan Chougale | B25 |

**Project guide:** Dr. Tanvi R. Patil

## Acknowledgements

- [FinGPT](https://github.com/AI4Finance-Foundation/FinGPT) for the financial forecaster LoRA adapter
- Meta AI for Llama-2
- Hugging Face `transformers`, `peft` and `bitsandbytes`

## Disclaimer

FutureWise is an academic prototype. Its output is machine-generated summarisation of public news and is **not financial advice**. Always do your own research before making investment decisions.
