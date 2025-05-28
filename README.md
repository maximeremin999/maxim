import json
import os
from fastapi import FastAPI, Request
from dotenv import load_dotenv
import aiohttp

load_dotenv()

app = FastAPI()
BOT_TOKEN = os.getenv("BOT_TOKEN")
ADMIN_ID = os.getenv("ADMIN_ID")  # ID для уведомлений

DATA_FILE = "db.json"

# Загружаем хранилище сообщений
if os.path.exists(DATA_FILE):
    with open(DATA_FILE, "r") as f:
        store = json.load(f)
else:
    store = {}

# Сохраняем хранилище
def save_store():
    with open(DATA_FILE, "w") as f:
        json.dump(store, f, ensure_ascii=False, indent=2)

# Отправка уведомления себе
async def send_message(text):
    url = f"https://api.telegram.org/bot{BOT_TOKEN}/sendMessage"
    async with aiohttp.ClientSession() as session:
        await session.post(url, json={"chat_id": ADMIN_ID, "text": text})

@app.post("/webhook")
async def webhook(request: Request):
    data = await request.json()

    if "business_message" in data:
        msg = data["business_message"]
        mid = str(msg["message_id"])
        text = msg.get("text", "[медиа]")
        store[mid] = text
        save_store()

    elif "edited_business_message" in data:
        msg = data["edited_business_message"]
        mid = str(msg["message_id"])
        new_text = msg.get("text", "[медиа]")
        old_text = store.get(mid, "[неизвестно]")
        await send_message(f"📌 Сообщение отредактировано:\n\nБыло: {old_text}\nСтало: {new_text}")
        store[mid] = new_text
        save_store()

    elif "deleted_business_messages" in data:
        for mid in data["deleted_business_messages"]:
            mid = str(mid)
            deleted = store.get(mid, "[неизвестно]")
            await send_message(f"❌ Сообщение удалено:\n\n{deleted}")
            if mid in store:
                del store[mid]
        save_store()

    return {"ok": True}