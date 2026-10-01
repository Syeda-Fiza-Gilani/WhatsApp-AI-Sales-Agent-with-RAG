# WhatsApp AI Sales Agent with RAG (n8n Workflow)

An n8n automation that answers customer messages on WhatsApp — text, voice notes, photos, and videos — using an AI sales agent that looks up answers in your product catalogue (RAG = Retrieval-Augmented Generation).

---

## What it does

1. A customer sends a message on WhatsApp — text, a voice note, a photo, or a video.
2. The workflow figures out what type of message it is.
3. Voice notes are transcribed to text. Photos are described by GPT Vision. Videos are described by Gemini.
4. Whatever the input type, it all becomes plain text.
5. An AI Sales Agent reads that text, searches your product catalogue (stored as a searchable knowledge base), and writes a factual, friendly reply.
6. The reply is sent back to the customer on WhatsApp automatically.
7. Each customer gets their own short-term memory, so the agent remembers the last few messages in that conversation.

If the message type isn't supported (e.g. a sticker or document), the customer gets a polite "please resend as text, voice, photo or video" message instead of the workflow breaking.

<img width="1184" height="684" alt="image" src="https://github.com/user-attachments/assets/883c1f60-1bf2-42f2-99a4-2d9ec3f9ce3f" />

---

## Two parts of this workflow

### Part 1 — Build the Knowledge Base (run manually, once, or whenever your catalogue changes)
`Populate Product Catalogue` (manual trigger) → `Knowledge Base Settings` → `Download Product Brochure` → `Extract Brochure Text` → `Split Brochure Text` → `Embeddings for Indexing` → `Insert Product Catalogue into Vector Store`

This downloads your product brochure/catalogue PDF, extracts the text, splits it into chunks, turns those chunks into embeddings, and stores them in an in-memory vector store the AI agent can search.

### Part 2 — Handle Live WhatsApp Messages (runs automatically)
`WhatsApp Trigger` → `Configuration` → `Route by Message Type` → (type-specific handling) → `Prepare Agent Input` → `AI Sales Agent` (with chat memory + catalogue search tool) → `Reply to Customer`

---

## Nodes at a glance

| Node | What it does |
|---|---|
| Populate Product Catalogue | Manual trigger to (re)build the knowledge base |
| Knowledge Base Settings | Holds the brochure URL, chunk size/overlap, and vector store key |
| Download Product Brochure | Downloads the catalogue PDF from a URL |
| Extract Brochure Text | Pulls plain text out of the PDF |
| Split Brochure Text / Embeddings for Indexing | Chunks the text and converts it to embeddings |
| Insert Product Catalogue into Vector Store | Saves the embeddings into an in-memory vector store |
| WhatsApp Trigger | Fires whenever a new WhatsApp message arrives |
| Configuration | One place to hold all settings — models used, prompts, phone number ID, reply length limit |
| Route by Message Type | Sends the message down the right path: Audio, Image, Video, Text, or Unsupported |
| Get [Audio/Image/Video] Media URL + Download [...] File | Fetches the actual media file from WhatsApp |
| Transcribe Voice Note | Converts voice notes to text (OpenAI Whisper) |
| Analyze Image with GPT Vision | Describes what's in a photo |
| Describe Video with Gemini | Describes what's happening in a video |
| Reply Unsupported Message | Polite fallback for message types we don't handle |
| Prepare Agent Input | Combines message type, text, caption, and sender into one clean object |
| AI Sales Agent | The LLM agent that reads the input and writes the reply |
| OpenAI Chat Model | The language model powering the agent |
| Per-Customer Chat Memory | Keeps a short rolling memory per customer phone number |
| query_product_catalogue / Product Catalogue (Retrieve) / Embeddings for Retrieval | Lets the agent search the catalogue knowledge base as a tool |
| Reply to Customer | Sends the final AI-written reply back on WhatsApp |

---

## What you need before running this

**Credentials to connect inside n8n** (none are included in this export — you must add your own):
- **WhatsApp Business API** credential (Meta/Facebook Developer app, with a verified phone number)
- **OpenAI API** credential (used for the chat model, Whisper transcription, GPT Vision, and embeddings)
- **Google Gemini API** credential (used for video understanding)

**Values to fill in** (search the JSON for `<__PLACEHOLDER_VALUE__...__>` or check these nodes):
- `Knowledge Base Settings` → `product_brochure_url`: a direct link to your product catalogue PDF
- `Configuration` → `phone_number_id`: your WhatsApp Business phone number ID from the Meta app dashboard

**Models used by default** (edit these in the `Configuration` node if you want different ones):
- `openai_model`: `gpt-5.4-mini`
- `vision_model`: `gpt-5.4-mini`
- `video_model`: `models/gemini-3.5-flash`

> Note: these model names come directly from the uploaded workflow. Double-check they're still valid/available in your OpenAI and Gemini accounts before going live — model names get renamed or deprecated over time.

---

## Setup steps

1. Import `whatsapp-ai-sales-agent-rag-NO-CREDENTIALS.json` into n8n.
2. Add your WhatsApp Business, OpenAI, and Google Gemini credentials in n8n, and attach them to the relevant nodes (WhatsApp nodes, OpenAI nodes, and the "Describe Video with Gemini" node).
3. Open `Knowledge Base Settings` and paste in your product brochure PDF URL.
4. Open `Configuration` and paste in your WhatsApp phone number ID.
5. Run `Populate Product Catalogue` manually once to build the knowledge base. Re-run it any time your catalogue changes (note: `clearStore` is on, so each run replaces the old catalogue data).
6. Activate the workflow so `WhatsApp Trigger` starts listening.
7. Send a test WhatsApp message (text first, then try a voice note, a photo, and a video) to confirm each path works.

---

## Good to know

- The in-memory vector store does not persist if your n8n instance restarts — for a production setup, swap it for a persistent vector store (e.g. Pinecone, Supabase, Qdrant).
- `Per-Customer Chat Memory` is keyed by the customer's WhatsApp number, so conversations don't mix between customers.
- `Reply to Customer` trims the AI's reply to `max_reply_length` characters (1500 by default) to avoid WhatsApp message limits.
