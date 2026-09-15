# Notes to PDF Automation

A free, self-hosted automation that turns a plain `.txt` file of class notes into a clean, styled PDF — automatically restructured, de-duplicated, and expanded with real-life context, without adding any new information beyond what you wrote.

## What it does

1. You upload a `.txt` file of notes through a simple web form.
2. An AI (Groq, free tier) reorganizes the notes into clear sections with headings, merges any duplicate points, and — for each existing concept only — adds:
   - a real-life application or example
   - the underlying logic/reasoning behind it
   - a practical tip for using it effectively
3. A "Key Takeaways" section is added at the end, summarizing only what's already covered.
4. The restructured notes are converted to styled HTML (light color theme, clean typography).
5. The HTML is rendered into a real PDF using Gotenberg (a self-hosted, open-source PDF conversion service).
6. The finished PDF is saved automatically to a folder on your computer.

No content is invented by the AI — it strictly reorganizes, de-duplicates, and contextualizes what's already in your notes.

## Tech stack (100% free)

- **[n8n](https://n8n.io)** — self-hosted automation platform (runs in Docker)
- **[Groq](https://console.groq.com)** — free-tier LLM API for text restructuring (model: `openai/gpt-oss-20b`)
- **[Gotenberg](https://gotenberg.dev)** — self-hosted, open-source HTML-to-PDF conversion (runs in its own Docker container)

## Setup

### 1. Prerequisites
- [Docker Desktop](https://www.docker.com/products/docker-desktop) installed and running
- A free [Groq](https://console.groq.com) account and API key

### 2. Run Gotenberg
```
docker run -d -p 3000:3000 gotenberg/gotenberg:8
```

### 3. Run n8n
Replace the output folder path with a real folder on your machine:
```
docker run -it --rm --name n8n -p 5678:5678 -v n8n_data:/home/node/.n8n -v "C:\path\to\your\output\folder:/files/output" -e N8N_RESTRICT_FILE_ACCESS_TO=/files/output docker.n8n.io/n8nio/n8n
```

### 4. Import the workflow
- Open n8n at `http://localhost:5678`
- Import `workflow.json` from this repo
- Add your Groq API key as a Header Auth credential (`Authorization: Bearer YOUR_KEY`) and attach it to the Groq HTTP Request node

### 5. Use it
- Open the form URL shown on the "On form submission" node
- Upload a `.txt` file of your notes
- Find your finished PDF in the output folder you mounted

## Notes

- Groq's free tier occasionally returns slightly inconsistent formatting, which the workflow handles gracefully with fallback parsing.
- This project intentionally keeps everything self-hosted and free — no paid APIs required.

## Challenges & Debugging (image binary handling)

Getting AI-generated illustrations properly embedded into the PDF turned into the hardest part of this project. A rough timeline of what went wrong and how it was fixed:

- **Broken image icons in the PDF**: Images generated fine, but the final PDF only showed broken/placeholder image icons instead of the actual pictures.
- **First suspect — node-to-node data loss**: n8n's "Loop Over Items" node and its `$node["NodeName"]` cross-node references turned out to be unreliable for passing binary (image) data across a loop boundary. Data referenced this way looked present in isolated tests but came back empty during a real end-to-end run.
- **Second suspect — the Markdown node**: Converting the AI's markdown text to HTML via n8n's built-in Markdown node silently dropped the binary image data attached to that item, since the node only preserves/transforms JSON fields.
- **Attempted fix — a Merge node**: Split the workflow into two parallel branches (one for markdown→HTML conversion, one carrying the raw binary untouched) and recombined them with a Merge node (Combine by Position). This got real image data flowing to the final step again.
- **Real root cause — filesystem binary storage**: Even with the data "present," the actual embedded output was still empty. Debug logging revealed n8n was storing binary data in **filesystem mode**, meaning `binary.data` held an internal reference string (e.g. `"filesystem-v2"`) rather than the actual base64 image content. The fix was using n8n's `this.helpers.getBinaryDataBuffer()` helper inside a Code node to properly resolve the real file content instead of reading `.data` directly.
- **Placement bug**: Once real images were finally embedding, they all landed at the end of the PDF instead of next to their relevant heading. This came down to two smaller bugs: a `slugify()` function that stripped out hyphens (while n8n's auto-generated heading IDs use hyphens), and a markdown-to-HTML converter that only recognized `#`, `##`, and `###` headings — missing any heading written with four or more `#` symbols by the AI.

After all of this, the image-generation + smart placement feature was ultimately shelved (see below) in favor of a simpler, reliable text-only PDF — the debugging above is kept here for reference and for anyone who wants to pick the feature back up.

## Future Improvements / TODO

This project currently ships as a **text-only** PDF generator. Image generation was built, debugged extensively, and then intentionally removed for reliability — it's a good next feature for anyone extending this project.

- [ ] **Re-add AI-selected illustrations**: Have the AI flag only genuinely hard-to-visualize concepts (not one image per heading) and generate a simple diagram for each via a free image API (e.g. Pollinations.ai).
- [ ] **Fix image placement**: Match each generated image to its actual heading reliably (previous attempt broke on AI wording mismatches between the flagged "concept" name and the actual heading text — may need fuzzy matching instead of exact slug matching).
- [ ] **Handle zero-image runs gracefully**: Some notes files won't need any illustrations at all — make sure the loop/pipeline doesn't stall when there's nothing to generate.
- [ ] **Better AI output reliability**: The free Groq model occasionally deviates from the requested output format — consider a stricter structured-output mode or a retry step.
- [ ] **Multiple file batch support**: Currently one file in, one PDF out — could be extended to batch-process a folder of notes.
- [ ] **Custom styling options**: Let the user pick a color theme/font via the form instead of a hardcoded style.
- [ ] **Handle very large notes files**: Add pagination or chunking for notes that exceed the AI model's context/token limits.

---

*Built with the help of Claude (Anthropic). Project started September 9, 2026.*
