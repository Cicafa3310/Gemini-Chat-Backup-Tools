# Gemini Chat Backup Tools

A small set of scripts to export and archive your Google Gemini conversation history — stored locally and ready to feed into tools like **NotebookLM**.

---

## 📦 What's included

| File | Where to run | Purpose |
|---|---|---|
| `gemini_live_export.js` | Individual chat page | Exports a single live Gemini chat |
| `gemini_chat_export.js` | myactivity.google.com | Exports chat history from Google Activity |
| `split_for_notebooklm.py` | Terminal | Splits large exports into NotebookLM-sized chunks |

---

## 🟡 `gemini_live_export.js` — Live Chat Exporter

Exports the full conversation from any open Gemini chat page directly to your Downloads folder. Scrolls up automatically to load the entire history before extracting.

### How to use

1. Open a Gemini chat: `https://gemini.google.com/app/XXXXXXXXXXXXXX`
2. Open DevTools — **F12** → **Console** tab
3. Paste the script below and press **Enter**
4. Wait — the script will scroll up to load all messages, then download a `.txt` file

### Script

```javascript
(async () => {
  const monthNames = ["January","February","March","April","May","June",
                      "July","August","September","October","November","December"];

  console.log("📜 Step 1/2 — Loading all messages...");

  const scroller = document.querySelector("infinite-scroller.chat-history");
  if (!scroller) { console.error("❌ Container not found!"); return; }

  let prev = 0, same = 0, totalScrolls = 0;

  while (same < 8) {
    scroller.scrollTop = 0;
    await new Promise(r => setTimeout(r, 2000));
    scroller.scrollTop = 300;
    await new Promise(r => setTimeout(r, 300));
    scroller.scrollTop = 0;
    await new Promise(r => setTimeout(r, 1000));

    let curr = document.querySelectorAll("user-query, model-response").length;
    totalScrolls++;
    if (curr === prev) { same++; console.log(`   Blocks: ${curr} — no change (${same}/8)`); }
    else { same = 0; prev = curr; console.log(`   Blocks: ${curr} — loading...`); }
  }

  console.log(`✅ All loaded: ${prev} blocks (${totalScrolls} scrolls)`);
  console.log("💬 Step 2/2 — Extracting...");

  const now = new Date();
  const results = [];
  const title = document.title?.replace(" - Gemini", "").trim() || "Gemini Chat";

  results.push(`EXPORT: ${title}`);
  results.push(`URL: ${location.href}`);
  results.push(`Exported: ${now.toLocaleString("en-GB")}`);
  results.push("");

  const allBlocks = document.querySelectorAll("user-query, model-response");
  let msgIndex = 0;

  allBlocks.forEach((block) => {
    const tag = block.tagName.toLowerCase();

    if (tag === "user-query") {
      const queryDiv = block.querySelector(".query-text");
      if (!queryDiv) return;
      const lines = Array.from(queryDiv.querySelectorAll("p.query-text-line"))
        .map(p => p.innerText.trim())
        .filter(t => t.length > 0 && t !== "Ваш запрос");
      const text = lines.join("\n").trim();
      if (!text) return;
      msgIndex++;
      results.push(`[${msgIndex}] USER: ${text}`);
      results.push("------------------------------------------");

    } else if (tag === "model-response") {
      const markdown = block.querySelector(".markdown");
      if (!markdown) return;
      const text = markdown.innerText.replace(/^Ответ Gemini\s*/i, "").trim();
      if (!text) return;
      results.push(`ANSWER:\n${text}`);
      results.push("==========================================");
      results.push("");
    }
  });

  const output = results.join("\n");
  const chatId = location.pathname.split("/").pop() || "chat";
  const filename = `gemini_chat_${chatId}_${now.toISOString().slice(0,10)}.txt`;

  const blob = new Blob([output], { type: "text/plain;charset=utf-8" });
  const a = document.createElement("a");
  a.href = URL.createObjectURL(blob);
  a.download = filename;
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);

  console.log(`✅ Done! ${filename}`);
  console.log(`   Messages: ${msgIndex} | Chars: ${output.length}`);
})();
```

### Output format

```
EXPORT:   My Chat Title
URL:      https://gemini.google.com/app/abc123...
Exported: 10/03/2026, 14:32:00

[1] USER: Hello, can you help me with...
------------------------------------------
ANSWER:
Of course! Here's what I suggest...
==========================================

[2] USER: What about...
------------------------------------------
ANSWER:
...
```

### Notes

- Works on `gemini.google.com/app/...` pages only
- Stops scrolling after 8 passes with no new content
- Output filename: `gemini_chat_<chatId>_<date>.txt`

---

## 🟢 `gemini_chat_export.js` — Activity History Exporter

Exports chat history from your Google Activity page.

> **Note:** This script only captures what's currently loaded on screen. Before running it, **scroll down manually to the bottom of the page** to load all entries — otherwise older history will be missing.

### How to use

1. Open `https://myactivity.google.com/product/gemini`
2. **Scroll to the bottom of the page** to load all history
3. Open DevTools — **F12** → **Console** tab
4. Paste the script below and press **Enter**

### Script

```javascript
(async () => {
  window.scrollTo(0, 0);
  await new Promise(r => setTimeout(r, 1500));

  const monthNames = ["January","February","March","April","May","June",
                      "July","August","September","October","November","December"];

  const now = new Date();
  const dateHeader = `${now.getDate()} ${monthNames[now.getMonth()]} ${now.getFullYear()}`;

  const results = [];
  const title = document.title?.replace(" - Gemini", "").trim() || "Gemini Chat";

  results.push(`EXPORT: ${title}`);
  results.push(`URL: ${location.href}`);
  results.push("");
  results.push(`\n========== ${dateHeader.toUpperCase()} ==========\n`);

  const allBlocks = document.querySelectorAll("user-query, model-response");
  console.log(`Found blocks: ${allBlocks.length}`);

  let msgIndex = 0;

  allBlocks.forEach((block) => {
    const tag = block.tagName.toLowerCase();

    if (tag === "user-query") {
      const queryDiv = block.querySelector(".query-text");
      if (!queryDiv) return;
      const lines = Array.from(queryDiv.querySelectorAll("p.query-text-line"))
        .map(p => p.innerText.trim())
        .filter(t => t.length > 0 && t !== "Ваш запрос");
      const text = lines.join("\n").trim();
      if (!text) return;
      msgIndex++;
      results.push(`[${msgIndex}] USER: ${text}`);
      results.push("------------------------------------------");

    } else if (tag === "model-response") {
      const markdown = block.querySelector(".markdown");
      if (!markdown) return;
      const text = markdown.innerText.replace(/^Ответ Gemini\s*/i, "").trim();
      if (!text) return;
      results.push(`ANSWER:\n${text}`);
      results.push("==========================================");
      results.push("");
    }
  });

  const output = results.join("\n");
  const chatId = location.pathname.split("/").pop() || "chat";
  const filename = `gemini_chat_${chatId}_${now.toISOString().slice(0,10)}.txt`;

  const blob = new Blob([output], { type: "text/plain;charset=utf-8" });
  const a = document.createElement("a");
  a.href = URL.createObjectURL(blob);
  a.download = filename;
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);

  console.log(`✅ Done! ${filename}`);
  console.log(`   Blocks: ${allBlocks.length} | Messages: ${msgIndex} | Chars: ${output.length}`);
})();
```

---

## 🐍 `split_for_notebooklm.py` — History Splitter

NotebookLM has a **~500k character limit per source**. This script splits a large export into properly-sized chunks.

### How to use

```bash
python split_for_notebooklm.py gemini_history_09_03_2026.txt
```

### Script

```python
#!/usr/bin/env python3
import sys
import os

MAX_CHARS = 400_000
SPLIT_ON  = "\n=========="

def split_file(input_path: str):
    print(f"📂 Reading: {input_path}")
    with open(input_path, "r", encoding="utf-8") as f:
        content = f.read()

    total_chars = len(content)
    print(f"   Total chars: {total_chars:,}")
    print(f"   Expected parts: ~{total_chars // MAX_CHARS + 1}")

    sections = content.split(SPLIT_ON)
    parts = []
    current_part = ""

    for i, section in enumerate(sections):
        chunk = (SPLIT_ON + section) if i > 0 else section
        if len(current_part) + len(chunk) > MAX_CHARS and current_part:
            parts.append(current_part)
            current_part = chunk
        else:
            current_part += chunk

    if current_part:
        parts.append(current_part)

    base_name = os.path.splitext(os.path.basename(input_path))[0]
    input_dir = os.path.dirname(os.path.abspath(input_path))
    output_dir = os.path.join(input_dir, f"{base_name}_parts")

    os.makedirs(output_dir, exist_ok=True)
    print(f"   📁 Output folder: {output_dir}")

    total_parts = len(parts)
    print(f"\n✂️  Splitting into {total_parts} parts:\n")

    for idx, part_text in enumerate(parts, 1):
        header = f"[PART {idx} of {total_parts} | {len(part_text):,} chars]\n"
        header += f"[Source: {os.path.basename(input_path)}]\n"
        header += "=" * 60 + "\n\n"

        out_name = f"part{idx:02d}_of{total_parts:02d}.txt"
        out_path = os.path.join(output_dir, out_name)

        with open(out_path, "w", encoding="utf-8") as f:
            f.write(header + part_text)

        print(f"   ✅ {out_name}  ({len(part_text):,} chars)")

    print(f"\n🎉 Done! {total_parts} files saved to: {output_dir}")
    print(f"\n📌 Upload each file to NotebookLM as a separate source.")
    print(f"   Recommended: one at a time, wait for each to finish processing.")

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Usage: python split_for_notebooklm.py <file.txt>")
        sys.exit(1)
    input_file = sys.argv[1]
    if not os.path.exists(input_file):
        print(f"❌ File not found: {input_file}")
        sys.exit(1)
    split_file(input_file)
```

### What it does

- Splits into parts of **~400k characters** each (safely under the 500k limit)
- Always splits at date boundaries — never mid-conversation
- Creates a dedicated folder next to your file:

```
gemini_history_09_03_2026_parts/
    part01_of29.txt
    part02_of29.txt
    ...
```

### Then in NotebookLM

Upload each `.txt` file as a **separate source**, one at a time.

---

## 💡 Recommended workflow

```
1. Export individual chats   →  gemini_live_export.js
2. Export activity history   →  myactivity.google.com + gemini_chat_export.js
3. Split for AI              →  split_for_notebooklm.py
4. Upload to NotebookLM      →  part by part
```

---

## 📋 Requirements

- **JS scripts** — any modern browser (Chrome, Edge, Firefox)
- **`split_for_notebooklm.py`** — Python 3.6+, no extra dependencies

---

## ⚠️ Disclaimer

These tools interact with the Gemini web interface using browser APIs. Google may update their page structure at any time, which could require selector updates in the scripts. Use at your own risk.
