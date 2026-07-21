# wikipediabreaker
This tool allows you to absolutely destroy Wikipedia with Google colab. have fun :)

Run this one first (dont worry if it throws an error, ignore it)
!pip install xhtml2pdf wikipedia-api deep-translator

now run this one. it has a text box. type in the name of the article you would like to see absolutely destroyed and hit enter. you will get an "snapshot" every 5 translations. or more, if thats what you desire
import os
import re
import random
import time
import json
import urllib.request
import urllib.parse
from concurrent.futures import ThreadPoolExecutor
from bs4 import BeautifulSoup, NavigableString

# ------------------- CONFIGURATION -------------------
TOTAL_HOPS = 100 #this controls how many times the thing is translated
SAVE_EVERY = 5 # lower this if you want more snapshots, raise it if you would perfer less interruptions
MAX_WORKERS = 10 # idk what this means but you might 
BATCH_SIZE = 40  # Packs 40 text fragments into a single network call to fly through long pages
AVAILABLE_LANGS = ["ja", "ko", "ar", "he", "is", "fi", "hu", "vi"]
# -----------------------------------------------------

def google_translate_batch(texts, from_lang, to_lang):
    """Translates a combined batch of text items in a single API request."""
    if not texts:
        return []

    # Filter empty or tiny fragments out of processing but keep placeholders
    clean_payloads = []
    for t in texts:
        clean_payloads.append(t.strip() if t.strip() else "")

    combined_string = " ||| ".join(clean_payloads)
    try:
        encoded_text = urllib.parse.quote(combined_string)
        url = f"https://translate.googleapis.com/translate_a/single?client=gtx&sl={from_lang}&tl={to_lang}&dt=t&q={encoded_text}"
        req = urllib.request.Request(url, headers={'User-Agent': 'Mozilla/5.0'})
        with urllib.request.urlopen(req, timeout=8) as response:
            raw_data = json.loads(response.read().decode('utf-8'))
            translated_chunks = [part[0] for part in raw_data[0] if part[0]]
            full_translated = "".join(translated_chunks)

            # Split back into individual strings
            parts = full_translated.split("|||")

            # Match spacing back to original items
            results = []
            for idx, original in enumerate(texts):
                if idx < len(parts):
                    p = parts[idx].strip()
                    lead = " " if original.startswith(" ") else ""
                    trail = " " if original.endswith(" ") else ""
                    results.append(f"{lead}{p}{trail}")
                else:
                    results.append(original)
            return results
    except Exception:
        return texts

def purge_foreign_remnants(text):
    if not text: return ""
    cleaned = re.sub(r'[\u4e00-\u9fff\u3040-\u309f\u30a0-\u30ff\uac00-\ud7af\u0600-\u06ff\u0590-\u05ff\u0400-\u04ff]+', '', text)
    return re.sub(r'\s+', ' ', cleaned)

def main_terminal_run():
    print("🌐 LIGHTSPEED CHAOS ENGINE v18.0 (UNIFIED CHUNK AGGREGATOR)")
    print("----------------------------------------------------------")

    raw_input_term = input("🔍 Enter a topic to search (e.g., List of Linux distributions): ").strip()
    if not raw_input_term: return

    timestamp = time.strftime("%Y%m%d_%H%M%S")
    safe_topic_name = re.sub(r'[^a-zA-Z0-9_]', '_', raw_input_term.lower())
    session_dir = f"sessions/{safe_topic_name}_heavy_page_{timestamp}"
    os.makedirs(session_dir, exist_ok=True)

    try:
        wiki_url = f"https://en.wikipedia.org/wiki/{urllib.parse.quote(raw_input_term.replace(' ', '_'))}"
        req = urllib.request.Request(wiki_url, headers={'User-Agent': 'Mozilla/5.0'})
        with urllib.request.urlopen(req) as response:
            raw_html = response.read().decode('utf-8')
    except Exception as e:
        print(f"❌ Target acquisition failed: {e}")
        return

    soup = BeautifulSoup(raw_html, 'html.parser')

    # Resolve assets
    for attr, tags in [('href', soup.find_all(href=True)), ('src', soup.find_all(src=True))]:
        for tag in tags:
            val = tag[attr]
            if val.startswith('//'): tag[attr] = 'https:' + val
            elif val.startswith('/'): tag[attr] = 'https://en.wikipedia.org' + val

    valid_containers = soup.find_all(['p', 'figcaption', 'td', 'th', 'li', 'span', 'a', 'h1', 'h2', 'h3'])
    text_leaf_mappings = []
    seen_strings = set()

    for container in valid_containers:
        if any(x in "".join(str(container.parent.get('class', ''))) for x in ["nav", "sidebar", "mw-jump-link"]):
            continue
        for content in container.contents:
            if isinstance(content, NavigableString) and len(content.strip()) > 1 and content not in seen_strings:
                text_leaf_mappings.append({'string_node': content, 'current_text': str(content)})
                seen_strings.add(content)

    total_fragments = len(text_leaf_mappings)
    print(f"🎯 Isolated {total_fragments} text fragments. Processing in batches of {BATCH_SIZE}...")

    current_lang = "en"

    for i in range(1, TOTAL_HOPS + 1):
        next_lang = random.choice([l for l in AVAILABLE_LANGS if l != current_lang])

        # Segment into chunk batches
        batches = [text_leaf_mappings[x:x+BATCH_SIZE] for x in range(0, total_fragments, BATCH_SIZE)]

        def process_batch(batch):
            texts_to_trans = [item['current_text'] for item in batch]
            return google_translate_batch(texts_to_trans, current_lang, next_lang)

        with ThreadPoolExecutor(max_workers=MAX_WORKERS) as executor:
            batch_results = list(executor.map(process_batch, batches))

        # Re-apply flattened values
        flat_idx = 0
        for batch_res in batch_results:
            for translated_text in batch_res:
                text_leaf_mappings[flat_idx]['current_text'] = translated_text
                flat_idx += 1

        print(f"🌀 Hop {i:02d} Complete ➔ Vector shifted into [{next_lang.upper()}]")
        current_lang = next_lang

        if i % SAVE_EVERY == 0 or i == TOTAL_HOPS:
            # Fork back to English in massive fast batches
            batches = [text_leaf_mappings[x:x+BATCH_SIZE] for x in range(0, total_fragments, BATCH_SIZE)]

            def fork_batch(batch):
                texts_to_trans = [item['current_text'] for item in batch]
                return google_translate_batch(texts_to_trans, current_lang, "en")

            with ThreadPoolExecutor(max_workers=MAX_WORKERS) as executor:
                fork_results = list(executor.map(fork_batch, batches))

            flat_idx = 0
            for batch_res in fork_results:
                for eng_text in batch_res:
                    clean_txt = purge_foreign_remnants(eng_text)
                    old_node = text_leaf_mappings[flat_idx]['string_node']
                    new_node = NavigableString(clean_txt)

                    if old_node.parent:
                        old_node.replace_with(new_node)
                        text_leaf_mappings[flat_idx]['string_node'] = new_node
                    flat_idx += 1

            html_filename = f"{session_dir}/hop_{i:03d}_{safe_topic_name}.html"
            with open(html_filename, "w", encoding="utf-8") as f:
                f.write(str(soup))
            print(f"   💾 [Snapshot Saved] ➔ '{html_filename}'")

    print(f"\n✨ Operation complete! Long article processed smoothly inside: '{session_dir}/'")

if __name__ == "__main__":
    main_terminal_run()
