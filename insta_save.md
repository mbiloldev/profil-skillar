# 📸 InstaBot — Instagram Video Saqlovchi Bot

## Loyiha haqida

Bu loyiha Python va `python-telegram-bot` kutubxonasi yordamida yaratilgan Telegram boti bo'lib, foydalanuvchi Instagram havolasini yuborganda videoni yuklab beradi.

---

## 📁 Fayl tuzilmasi

```
instabot/
├── bot.py              # Asosiy bot fayli
├── downloader.py       # Instagram video yuklovchi modul
├── config.py           # Token va sozlamalar
├── requirements.txt    # Kerakli kutubxonalar
└── README.md
```

---

## ⚙️ Texnologiyalar

| Kutubxona | Maqsad |
|---|---|
| `python-telegram-bot` | Telegram Bot API bilan ishlash |
| `yt-dlp` | Instagram videolarini yuklab olish |
| `python-dotenv` | Token va sirlarni `.env` faylida saqlash |

---

## 🛠️ O'rnatish

### 1. Kerakli kutubxonalarni o'rnatish

```bash
pip install python-telegram-bot yt-dlp python-dotenv
```

### 2. `.env` fayl yaratish

```
BOT_TOKEN=your_telegram_bot_token_here
```

### 3. Bot tokenini olish

[@BotFather](https://t.me/BotFather) ga o'tib yangi bot yarating va tokenni oling.

---

## 📄 Kod

### `config.py`

```python
import os
from dotenv import load_dotenv

load_dotenv()

BOT_TOKEN = os.getenv("BOT_TOKEN")
DOWNLOAD_PATH = "downloads/"
```

---

### `downloader.py`

```python
import yt_dlp
import os
from config import DOWNLOAD_PATH

def download_instagram_video(url: str) -> str | None:
    """
    Instagram URL dan video yuklab oladi.
    Muvaffaqiyatli bo'lsa fayl yo'lini qaytaradi,
    xato bo'lsa None qaytaradi.
    """
    os.makedirs(DOWNLOAD_PATH, exist_ok=True)

    ydl_opts = {
        "outtmpl": f"{DOWNLOAD_PATH}%(id)s.%(ext)s",
        "format": "mp4",
        "quiet": True,
        "noplaylist": True,
    }

    try:
        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            info = ydl.extract_info(url, download=True)
            filename = ydl.prepare_filename(info)
            return filename
    except Exception as e:
        print(f"Xato: {e}")
        return None
```

---

### `bot.py`

```python
import os
import logging
from telegram import Update
from telegram.ext import (
    ApplicationBuilder,
    CommandHandler,
    MessageHandler,
    filters,
    ContextTypes,
)
from config import BOT_TOKEN
from downloader import download_instagram_video

logging.basicConfig(level=logging.INFO)

# /start komandasi
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "👋 Salom! Men Instagram video yuklovchi botman.\n\n"
        "📎 Instagram post yoki reel havolasini yuboring — videoni yuklab beraman!"
    )

# Xabar qabul qiluvchi
async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    url = update.message.text.strip()

    # Instagram URL tekshiruvi
    if "instagram.com" not in url:
        await update.message.reply_text("❗ Iltimos, to'g'ri Instagram havolasini yuboring.")
        return

    wait_msg = await update.message.reply_text("⏳ Video yuklanmoqda, iltimos kuting...")

    filepath = download_instagram_video(url)

    if filepath and os.path.exists(filepath):
        await update.message.reply_text("✅ Video topildi, yuborilmoqda...")
        with open(filepath, "rb") as video_file:
            await update.message.reply_video(video=video_file)
        os.remove(filepath)  # Yuklab bo'lgach faylni o'chirish
    else:
        await update.message.reply_text(
            "❌ Video yuklab bo'lmadi. Sabablari:\n"
            "• Havola noto'g'ri\n"
            "• Post yopiq (private) bo'lishi mumkin\n"
            "• Instagram vaqtincha blok qilgan bo'lishi mumkin"
        )

    await wait_msg.delete()

# Botni ishga tushirish
def main():
    app = ApplicationBuilder().token(BOT_TOKEN).build()

    app.add_handler(CommandHandler("start", start))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))

    print("🤖 Bot ishga tushdi...")
    app.run_polling()

if __name__ == "__main__":
    main()
```

---

## 🚀 Botni ishga tushirish

```bash
python bot.py
```

---

## 📌 Muhim eslatmalar

- **Private (yopiq) akkauntlar** — bunday postlarni yuklab bo'lmaydi, faqat ochiq postlar ishlaydi.
- **Cookies** — agar Instagram bloklasa, `yt-dlp` ga Instagram cookies qo'shish kerak bo'ladi:
  ```python
  ydl_opts["cookiefile"] = "instagram_cookies.txt"
  ```
- **Fayl hajmi** — Telegram orqali maksimal 50 MB gacha video yuborish mumkin. Katta hajmli videolar uchun `reply_document()` ishlatish tavsiya etiladi.
- **Rate limit** — juda ko'p so'rov yuborilsa Instagram vaqtincha IP ni bloklashi mumkin.

---

## 🔒 Huquqiy maslahat

Bu bot faqat shaxsiy foydalanish uchun mo'ljallangan. Boshqalarning kontentini ruxsatsiz tarqatish mualliflik huquqlarini buzishi mumkin.

---

## 📬 Kengaytirish g'oyalari

- [ ] Foydalanuvchi tarixi (history) — oxirgi 10 ta yuklangan video
- [ ] Audio (MP3) sifatida yuklash imkoniyati
- [ ] Reels, Stories va IGTV qo'llab-quvvatlash
- [ ] Admin panel (statistika ko'rish)
- [ ] Webhook orqali serverga deploy qilish
