"""
Itihas AI - History Facts Agent
--------------------------------
Roz 20 history fact + caption + image generate karta hai,
aur Google Drive me date-wise folder me numbered posts (post_01, post_02...) save karta hai.
(Koi auto-posting nahi - sirf content ready karta hai.)
"""

import os
import io
import time
import datetime
import requests
from groq import Groq
from google.oauth2.credentials import Credentials
from googleapiclient.discovery import build
from googleapiclient.http import MediaIoBaseUpload

# ---------- CONFIG (GitHub Secrets se aayega, yaha khali chhodo) ----------
GROQ_API_KEY = os.environ["GROQ_API_KEY"]
DRIVE_PARENT_FOLDER_ID = os.environ["DRIVE_PARENT_FOLDER_ID"]  # jis folder ke andar date-folders banenge
OAUTH_CLIENT_ID = os.environ["261012966066-besqecupica72fhuvnne9v7ifr846fqk.apps.googleusercontent.com"]
OAUTH_CLIENT_SECRET = os.environ["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"]
OAUTH_REFRESH_TOKEN = os.environ["OAUTH_REFRESH_TOKEN"]

MAX_RETRIES = 3
POSTS_PER_DAY = 20
DELAY_BETWEEN_POSTS_SECONDS = 5  # Gemini/Image API ko rate-limit se bachane ke liye

# ---------- STEP 1: Gemini se History Fact + Caption Generate ----------

def build_style_prompt(already_used_hooks):
    avoid_block = ""
    if already_used_hooks:
        joined = "\n".join(f"- {h}" for h in already_used_hooks)
        avoid_block = f"\nYeh hooks pehle hi use ho chuke hain aaj, inhe REPEAT mat karna aur inse alag topic/angle chuno:\n{joined}\n"

    return f"""
Tum "Itihas AI" channel ke liye content likhte ho - Hindi POV history shorts.
Ek REAL, VERIFIABLE Indian history fact do jisme:
1. Koi wrongdoing/scandal/chhupaya gaya sach ka angle ho
2. Viewer ki personal zindagi (health/family/paisa/pehchan) se touch ho
3. First-person ya dramatic POV tone ho
{avoid_block}
Format bilkul yeh do (aur kuch mat likho):
HOOK: [1-2 line dramatic hook, Hindi me]
FACT: [3-4 line factual explanation, Hindi me]
CAPTION: [Instagram caption with 5-6 relevant hashtags]
"""


def generate_content_with_retry(client, already_used_hooks):
    prompt = build_style_prompt(already_used_hooks)

    for attempt in range(1, MAX_RETRIES + 1):
        response = client.chat.completions.create(
            model="llama-3.3-70b-versatile",
            messages=[{"role": "user", "content": prompt}],
        )
        text = response.choices[0].message.content.strip()

        # Quality check loop - dekho HOOK/FACT/CAPTION teeno present hain ya nahi
        if "HOOK:" in text and "FACT:" in text and "CAPTION:" in text:
            return text
        print(f"  Attempt {attempt}: format sahi nahi tha, dobara try kar raha hoon...")
        time.sleep(3)

    # 3 baar fail ho to jo bhi mila wahi use karo
    return text


def parse_content(raw_text):
    parts = {"hook": "", "fact": "", "caption": ""}
    current = None
    for line in raw_text.splitlines():
        if line.startswith("HOOK:"):
            current = "hook"
            parts["hook"] = line.replace("HOOK:", "").strip()
        elif line.startswith("FACT:"):
            current = "fact"
            parts["fact"] = line.replace("FACT:", "").strip()
        elif line.startswith("CAPTION:"):
            current = "caption"
            parts["caption"] = line.replace("CAPTION:", "").strip()
        elif current:
            parts[current] += " " + line.strip()
    return parts


# ---------- STEP 2: Free Image Generate (Pollinations.ai - no API key needed) ----------

def generate_image(hook_text):
    prompt = f"cinematic historical illustration, dramatic lighting, {hook_text}, no text overlay"
    url = f"https://image.pollinations.ai/prompt/{requests.utils.quote(prompt)}"

    for attempt in range(1, MAX_RETRIES + 1):
        r = requests.get(url, timeout=60)
        if r.status_code == 200 and len(r.content) > 1000:
            return r.content
        print(f"  Attempt {attempt}: image generation fail hua, dobara try...")
        time.sleep(3)

    raise Exception("Image generate nahi ho payi 3 tries ke baad")


# ---------- STEP 3: Google Drive Upload ----------

def get_drive_service():
    creds = service_account.Credentials.from_service_account_file(
        SERVICE_ACCOUNT_JSON_PATH,
        scopes=["https://www.googleapis.com/auth/drive"],
    )
    return build("drive", "v3", credentials=creds)


def create_dated_folder(drive_service, date_str):
    metadata = {
        "name": date_str,
        "mimeType": "application/vnd.google-apps.folder",
        "parents": [DRIVE_PARENT_FOLDER_ID],
    }
    folder = drive_service.files().create(body=metadata, fields="id").execute()
    return folder["id"]


def upload_file(drive_service, folder_id, filename, content_bytes, mime_type):
    for attempt in range(1, MAX_RETRIES + 1):
        try:
            media = MediaIoBaseUpload(io.BytesIO(content_bytes), mimetype=mime_type)
            metadata = {"name": filename, "parents": [folder_id]}
            drive_service.files().create(body=metadata, media_body=media).execute()
            return
        except Exception as e:
            print(f"  Attempt {attempt}: upload fail - {e}")
            time.sleep(3)
    raise Exception(f"{filename} upload nahi ho payi 3 tries ke baad")


# ---------- MAIN ----------

def main():
    today = datetime.date.today().isoformat()  # jaise 2026-07-04
    print(f"Aaj ke {POSTS_PER_DAY} history-fact posts bana raha hoon: {today}")

    client = Groq(api_key=GROQ_API_KEY)

    drive_service = get_drive_service()
    folder_id = create_dated_folder(drive_service, today)

    already_used_hooks = []

    for i in range(1, POSTS_PER_DAY + 1):
        post_num = str(i).zfill(2)  # 01, 02, ... 20
        print(f"\n--- Post {post_num}/{POSTS_PER_DAY} ---")

        raw = generate_content_with_retry(client, already_used_hooks)
        parts = parse_content(raw)
        already_used_hooks.append(parts["hook"])
        print("Content generate ho gaya:", parts["hook"])

        image_bytes = generate_image(parts["hook"])
        print("Image generate ho gayi.")

        caption_text = f"HOOK: {parts['hook']}\n\nFACT: {parts['fact']}\n\nCAPTION: {parts['caption']}"
        upload_file(drive_service, folder_id, f"post_{post_num}_caption.txt", caption_text.encode("utf-8"), "text/plain")
        upload_file(drive_service, folder_id, f"post_{post_num}_image.jpg", image_bytes, "image/jpeg")

        print(f"Post {post_num} Drive me daal diya gaya.")

        if i < POSTS_PER_DAY:
            time.sleep(DELAY_BETWEEN_POSTS_SECONDS)  # rate-limit se bachne ke liye

    print(f"\nSab {POSTS_PER_DAY} posts Google Drive me daal diye gaye: folder '{today}'")


if __name__ == "__main__":
    main()
