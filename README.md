# Gemini Chat Backup Tools

A small set of scripts to export and archive your Google Gemini conversation history — stored locally and ready to feed into tools like **NotebookLM**.

---

## 📦 What's included

| File | Where to run | Purpose |
|---|---|---|
| [`gemini_live_export.js`](#-gemini_live_exportjs--live-chat-exporter) | Individual chat page | Exports a single live Gemini chat |
| [`gemini_chat_export.js`](#-gemini_chat_exportjs--activity-history-exporter) | myactivity.google.com | Exports chat history from Google Activity |
| [`split_for_notebooklm.py`](#-split_for_notebooklmpy--history-splitter) | Terminal | Splits large exports into NotebookLM-sized chunks |

---

## 🟡 `gemini_live_export.js` — Live Chat Exporter

Exports the full conversation from any open Gemini chat page directly to your Downloads folder. **Scrolls automatically** to load the entire history before extracting — no manual scrolling needed.

> **Heads up:** Loading takes ~3–4 seconds per scroll pass. For long chats (100+ messages) expect 1–2 minutes before the file downloads.

### How to use

1. Open a Gemini chat: `https://gemini.google.com/app/XXXXXXXXXXXXXX`
2. Open DevTools — **F12** → **Console** tab
3. Paste the script below and press **Enter**
4. Watch the console — it will log progress while scrolling, then download the file automatically

### Script

```javascript
(async () => {
  const monthNames = ["January","February","March","April","May","June",
                      "July","August","September","October","November","December"];

  console.log("📜 Step 1/2 — Loading all messages...");
  console.log("   ⏳ This may take 1–2 min for long chats. Watch the block count below.");

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
- Stops scrolling after 8 consecutive passes with no new content
- Output filename: `gemini_chat_<chatId>_<date>.txt`

---

## 🟢 `gemini_chat_export.js` — Activity History Exporter

Exports chat history from your Google Activity page.

> **Heads up:** The script scrolls down automatically to load all cards first. Speed depends on your connection — it polls for each modal instead of waiting a fixed delay. Expect roughly **~5 min for 300 dialogs, ~35 min for 3000+**. Remaining time is shown in the console after each dialog.

### How to use

1. Open `https://myactivity.google.com/product/gemini`
2. Open DevTools — **F12** → **Console** tab
3. Paste the script below and press **Enter**
4. Answer the setup prompts, then watch the console

### Script

```javascript
(async () => {
  const monthNames = ["January","February","March","April","May","June",
                      "July","August","September","October","November","December"];

  // ── Setup prompts (shown before anything starts) ───────────
  const scrollAns = prompt("📜 Setup 1/3 — Scroll limit\nHow many cards to load max?\n(leave empty = scroll to the very end)");
  const SCROLL_LIMIT = scrollAns && scrollAns.trim() !== "" ? parseInt(scrollAns) : null;

  const dateAns = prompt("📅 Setup 2/3 — Date filter\nExport only from this date onwards?\nFormat: DD.MM.YYYY  (e.g. 01.01.2025)\n(leave empty = export everything)");
  let fromDate = null;
  if (dateAns && dateAns.trim() !== "") {
    const p = dateAns.trim().split(".");
    if (p.length === 3) fromDate = new Date(parseInt(p[2]), parseInt(p[1]) - 1, parseInt(p[0]));
  }

  const exportAns = prompt("📦 Setup 3/3 — Export limit\nHow many dialogs to export?\n(leave empty = all loaded cards)");
  const EXPORT_LIMIT = exportAns && exportAns.trim() !== "" ? parseInt(exportAns) : null;

  const WAIT_MS = 100; // ms per poll tick — increase to 200-300 on slow connection

  // ── Step 1/2: Scroll DOWN to load cards ───────────────────
  console.log("📜 Step 1/2 — Loading activity cards...");
  if (SCROLL_LIMIT) console.log(`   ⏹ Will stop after ${SCROLL_LIMIT} cards`);
  if (fromDate)     console.log(`   📅 Date filter: from ${fromDate.toLocaleDateString("en-GB")}`);

  let prev = 0, same = 0, totalScrolls = 0;
  const scrollStart = Date.now();
  let dateStopReached = false;
  let lastDateCheck = 0;

  // Helper: check date of last loaded card without opening anything
  function checkLastCardDate() {
    const allCards = document.querySelectorAll('[jsname="MFYZYe"]');
    if (!allCards.length) return null;
    const lastCard = allCards[allCards.length - 1];
    const cwiz = lastCard.closest('[data-date]') || lastCard.querySelector('[data-date]');
    const raw = cwiz?.getAttribute('data-date');
    if (!raw || raw.length !== 8) return null;
    return new Date(parseInt(raw.slice(0,4)), parseInt(raw.slice(4,6))-1, parseInt(raw.slice(6,8)));
  }

  while (same < 8) {
    window.scrollTo(0, document.body.scrollHeight);
    await new Promise(r => setTimeout(r, 2000));
    window.scrollBy(0, -300);
    await new Promise(r => setTimeout(r, 300));
    window.scrollTo(0, document.body.scrollHeight);
    await new Promise(r => setTimeout(r, 1000));

    let curr = document.querySelectorAll('[jsname="MFYZYe"]').length;
    const elapsed = Math.round((Date.now() - scrollStart) / 1000);
    totalScrolls++;
    if (curr === prev) {
      same++;
      console.log(`   Cards: ${curr} — no change (${same}/8) | ${elapsed}s elapsed`);
    } else {
      same = 0; prev = curr;
      console.log(`   Cards: ${curr} — loading... | ${elapsed}s elapsed`);
    }
    if (SCROLL_LIMIT && curr >= SCROLL_LIMIT) {
      console.log(`   ⏹ Scroll limit reached: ${curr} cards`);
      break;
    }
    // ── Date check every 100 cards ──────────────────────────
    if (fromDate && curr >= lastDateCheck + 100) {
      lastDateCheck = curr;
      console.log(`   📅 Checking date at card ${curr}...`);
      const cardDate = checkLastCardDate();
      if (cardDate) {
        console.log(`   📅 Last card date: ${cardDate.toLocaleDateString("en-GB")}`);
        if (cardDate < fromDate) {
          console.log(`   ✅ Passed the target date — stopping scroll early!`);
          dateStopReached = true;
          break;
        }
      }
    }
  }

  console.log(`✅ All cards loaded: ${prev} total (${totalScrolls} scrolls)`);

  // ── Step 2/2: Extract conversations ───────────────────────
  console.log("💬 Step 2/2 — Extracting conversations...");

  function parseDate(raw) {
    if (!raw || raw.length !== 8) return null;
    const y = raw.slice(0, 4), m = parseInt(raw.slice(4, 6)), d = parseInt(raw.slice(6, 8));
    return `${d} ${monthNames[m - 1]} ${y}`;
  }

  function parseDateObj(raw) {
    if (!raw || raw.length !== 8) return null;
    return new Date(parseInt(raw.slice(0, 4)), parseInt(raw.slice(4, 6)) - 1, parseInt(raw.slice(6, 8)));
  }

  function closeAllModals() {
    const modals = document.querySelectorAll('.Wi5i2');
    let closed = 0;
    for (const m of modals) {
      const btn = Array.from(m.querySelectorAll('button i'))
        .find(i => i.textContent.trim() === 'close')?.closest('button');
      if (btn) { btn.click(); closed++; }
    }
    return closed;
  }

  let allLinks = Array.from(document.querySelectorAll('a.WFTFcf'));

  // Apply date filter
  if (fromDate) {
    allLinks = allLinks.filter(link => {
      const cwiz = link.closest('[data-date]');
      const d = parseDateObj(cwiz?.getAttribute('data-date'));
      return d && d >= fromDate;
    });
    console.log(`   📅 After date filter: ${allLinks.length} dialogs`);
  }

  const links = EXPORT_LIMIT ? allLinks.slice(0, EXPORT_LIMIT) : allLinks;
  const totalLinks = links.length;
  const secPerItem = 1.0;
  const estSec = Math.round(totalLinks * secPerItem);
  const estMin = Math.floor(estSec / 60);
  const estRemSec = estSec % 60;
  console.log(`   Found ${allLinks.length} dialogs, exporting ${totalLinks}`);
  console.log(`   ⏱ Estimated time: ~${estMin > 0 ? estMin + " min " : ""}${estRemSec} sec`);

  const now = new Date();
  let results = [];
  let lastDate = null;

  results.push(`EXPORT: Google Gemini Activity`);
  results.push(`URL: ${location.href}`);
  results.push(`Exported: ${now.toLocaleString("en-GB")}`);
  results.push(`Dialogs: ${totalLinks}${EXPORT_LIMIT ? ` (limited from ${allLinks.length})` : ""}${fromDate ? ` | From: ${fromDate.toLocaleDateString("en-GB")}` : ""}`);
  results.push("");

  for (let i = 0; i < links.length; i++) {
    const link = links[i];
    const card = link.closest('[jsname="MFYZYe"]');
    const cwiz = link.closest('[data-date]');
    const dateLabel = parseDate(cwiz?.getAttribute('data-date'));

    if (dateLabel && dateLabel !== lastDate) {
      results.push(`\n========== ${dateLabel.toUpperCase()} ==========\n`);
      lastDate = dateLabel;
    }

    const queryText = card?.querySelector('[jsname="r4nke"]')?.innerText
      .replace(/^Отправлен запрос\s*/i, '').replace(/^Sent to\s*/i, '').trim() || '?';
    let timeText = card?.querySelector('.H3Q9vf')?.innerText.split('•')[0].trim() || '';

    const remaining = totalLinks - i - 1;
    const remSec = Math.round(remaining * secPerItem);
    const remMin = Math.floor(remSec / 60);
    const remSecDisplay = remSec % 60;
    console.log(`   [${i + 1}/${totalLinks}] ${queryText.slice(0, 50)} — ~${remMin > 0 ? remMin + "m " : ""}${remSecDisplay}s left`);

    // ── Checkpoint every 20: force-close any lingering modals ─
    if (i > 0 && i % 20 === 0) {
      const closed = closeAllModals();
      await new Promise(r => setTimeout(r, WAIT_MS * 5));
      console.log(`   🔍 Checkpoint [${i}/${totalLinks}] — modals cleaned up (${closed} closed)`);
    }

    // ── Before clicking: make sure no modal is already open ───
    const stale = document.querySelector('.Wi5i2');
    if (stale) {
      closeAllModals();
      await new Promise(r => setTimeout(r, WAIT_MS * 4));
    }

    link.click();

    // Wait for new modal to appear
    let modal = null;
    for (let t = 0; t < 30; t++) {
      await new Promise(r => setTimeout(r, WAIT_MS));
      modal = document.querySelector('.Wi5i2');
      if (modal) break;
    }

    let answerText = '[not loaded]';

    if (modal) {
      const answerDiv = modal.querySelector('[jsname="buaNG"]');
      if (answerDiv) answerText = answerDiv.innerText.trim();
      if (!timeText) timeText = modal.querySelector('.jiqmSb')?.innerText.trim() || '??:??';

      const closeBtn = Array.from(modal.querySelectorAll('button i'))
        .find(i => i.textContent.trim() === 'close')?.closest('button');
      if (closeBtn) closeBtn.click();
      await new Promise(r => setTimeout(r, WAIT_MS * 3));
    }

    if (!timeText) timeText = '??:??';
    results.push(`[${timeText}] USER: ${queryText}\nANSWER:\n${answerText}\n------------------------------------------`);
  }

  const day   = String(now.getDate()).padStart(2, '0');
  const month = String(now.getMonth() + 1).padStart(2, '0');
  const year  = now.getFullYear();
  const filename = `gemini_history_${day}_${month}_${year}.txt`;

  const output = results.join('\n');
  const blob = new Blob([output], { type: 'text/plain' });
  const a = document.createElement('a');
  a.href = URL.createObjectURL(blob);
  a.download = filename;
  a.click();

  console.log(`✅ Done! File saved as: ${filename}`);
  console.log(`   Dialogs: ${totalLinks} | Characters: ${output.length}`);
})();
```

### Output format

```
EXPORT:   Google Gemini Activity
URL:      https://myactivity.google.com/product/gemini
Exported: 11/03/2026, 14:32:00
Dialogs:  300 (limited from 3366) | From: 01.01.2025

========== 9 MARCH 2026 ==========

[19:15] USER: Your message here...
ANSWER:
Gemini's response here...
------------------------------------------

========== 8 MARCH 2026 ==========

[14:03] USER: Another message...
ANSWER:
...
```

### Notes

- Works on `myactivity.google.com/product/gemini` only
- Three setup prompts appear before anything starts — scroll limit, date filter, export limit
- Every 20 dialogs: automatically checks and closes any lingering modals
- Before each dialog: waits for previous modal to fully close before opening next
- On slow connections, increase `WAIT_MS` from `100` to `200`–`300`
- Output filename: `gemini_history_DD_MM_YYYY.txt`

---

## 🐍 `split_for_notebooklm.py` — History Splitter

NotebookLM has a **~500k character limit per source**. This script splits a large export into properly-sized chunks.

### How to use

```bash
python split_for_notebooklm.py gemini_history_[your_name].txt
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
gemini_history_DD_MM_YYYY_parts/
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
