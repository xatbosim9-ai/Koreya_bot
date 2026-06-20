# -*- coding: utf-8 -*-
"""
KOREYA-GA so'z botı
====================
Bu fayl Telegram botning "miyasi". U words.json fayldan barcha
koreyscha so'zlarni o'qiydi va foydalanuvchiga 3 xil rejimda ko'rsatadi:
  1) Mavzular bo'yicha ko'rish (Boblar)
  2) Flashcard (kartochka) o'yini
  3) Qidiruv

Eslatma: Token (botning "parol"i) xavfsizlik uchun shu faylga emas,
balki muhit o'zgaruvchisiga (environment variable) yoziladi.
Pastdagi "BOT_TOKEN" qismini ko'ring.
"""

import json
import logging
import os
import random

from telegram import InlineKeyboardButton, InlineKeyboardMarkup, Update, WebAppInfo
from telegram.constants import ParseMode
from telegram.ext import (
    Application,
    CallbackQueryHandler,
    CommandHandler,
    ContextTypes,
    MessageHandler,
    filters,
)

# ---------------------------------------------------------------
# 1) SOZLAMALAR
# ---------------------------------------------------------------

BOT_TOKEN = os.environ.get("BOT_TOKEN", "BU_YERGA_TOKENINGIZNI_QOYING")

# Saytingiz manzili (Mini App). GitHub Pages'dan olingan https:// havola.
WEBAPP_URL = os.environ.get("WEBAPP_URL", "https://xatbosim9-ai.github.io/Koreya_bot/")

logging.basicConfig(
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    level=logging.INFO,
)
logger = logging.getLogger(__name__)

# ---------------------------------------------------------------
# 2) MA'LUMOTLARNI YUKLASH (words.json)
# ---------------------------------------------------------------

DATA_PATH = os.path.join(os.path.dirname(os.path.abspath(__file__)), "words.json")
with open(DATA_PATH, encoding="utf-8") as f:
    CHAPTERS = json.load(f)

ALL_WORDS = []
for ci, ch in enumerate(CHAPTERS):
    for wi, w in enumerate(ch["words"]):
        ALL_WORDS.append({**w, "chapter_idx": ci, "word_idx": wi, "chapter_title": ch["titleUz"]})

POS_EMOJI = {
    "noun": "🔵 Ot",
    "verb": "🔴 Fe'l",
    "adj": "🟣 Sifat",
    "adv": "🟡 Ravish",
    "idiom": "🟠 Ibora",
}


# ---------------------------------------------------------------
# 3) YORDAMCHI FUNKSIYALAR (matn va tugmalarni tayyorlash)
# ---------------------------------------------------------------

def main_menu_kb():
    """Bosh menyu tugmalari."""
    keyboard = [
        [InlineKeyboardButton("🌐 Saytni ochish", web_app=WebAppInfo(url=WEBAPP_URL))],
        [InlineKeyboardButton("📚 Mavzular", callback_data="menu:chapters")],
        [InlineKeyboardButton("🎴 Flashcard", callback_data="flash:start")],
        [InlineKeyboardButton("🔍 Qidirish", callback_data="menu:search")],
    ]
    return InlineKeyboardMarkup(keyboard)


def chapters_list_kb():
    """Barcha 12 ta mavzu ro'yxati tugma sifatida."""
    keyboard = []
    for i, ch in enumerate(CHAPTERS):
        label = f"{i+1:02d}. {ch['title']} ({len(ch['words'])} so'z)"
        keyboard.append([InlineKeyboardButton(label, callback_data=f"chapter:{i}:0")])
    keyboard.append([InlineKeyboardButton("🏠 Bosh menyu", callback_data="menu:main")])
    return InlineKeyboardMarkup(keyboard)


def word_card_text(ch_idx, w_idx):
    """Bitta so'z kartochkasining matnini tayyorlaydi."""
    ch = CHAPTERS[ch_idx]
    w = ch["words"][w_idx]
    pos = POS_EMOJI.get(w["posClass"], w["pos"])
    text = (
        f"*{ch['title']}* — {w_idx+1}/{len(ch['words'])}\n\n"
        f"🇰🇷 *{w['kr']}*  ({pos})\n"
        f"🇺🇿 {w['uz']}\n\n"
        f"_Misol:_\n"
        f"{w['exKr']}\n"
        f"{w['exUz']}"
    )
    return text


def word_card_kb(ch_idx, w_idx):
    """So'z kartochkasi ostidagi navigatsiya tugmalari (Oldingi/Keyingi)."""
    n = len(CHAPTERS[ch_idx]["words"])
    prev_i = (w_idx - 1) % n
    next_i = (w_idx + 1) % n
    keyboard = [
        [
            InlineKeyboardButton("⬅️ Oldingi", callback_data=f"chapter:{ch_idx}:{prev_i}"),
            InlineKeyboardButton("Keyingi ➡️", callback_data=f"chapter:{ch_idx}:{next_i}"),
        ],
        [InlineKeyboardButton("📚 Mavzular ro'yxati", callback_data="menu:chapters")],
        [InlineKeyboardButton("🏠 Bosh menyu", callback_data="menu:main")],
    ]
    return InlineKeyboardMarkup(keyboard)


def flash_front_kb():
    keyboard = [
        [InlineKeyboardButton("🔄 Javobni ko'rsatish", callback_data="flash:flip")],
        [InlineKeyboardButton("🔀 Aralashtirish", callback_data="flash:shuffle")],
        [InlineKeyboardButton("🏠 Bosh menyu", callback_data="menu:main")],
    ]
    return InlineKeyboardMarkup(keyboard)


def flash_back_kb():
    keyboard = [
        [InlineKeyboardButton("➡️ Keyingi karta", callback_data="flash:next")],
        [InlineKeyboardButton("🔀 Aralashtirish", callback_data="flash:shuffle")],
        [InlineKeyboardButton("🏠 Bosh menyu", callback_data="menu:main")],
    ]
    return InlineKeyboardMarkup(keyboard)


async def safe_edit(query, text, kb):
    """Xabarni tahrirlaydi; Markdown xato bersa, oddiy matn bilan urinib ko'radi."""
    try:
        await query.edit_message_text(text, reply_markup=kb, parse_mode=ParseMode.MARKDOWN)
    except Exception:
        await query.edit_message_text(text, reply_markup=kb)


# ---------------------------------------------------------------
# 4) BUYRUQLAR (/start)
# ---------------------------------------------------------------

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data["awaiting_search"] = False
    text = (
        "안녕하세요! 👋\n\n"
        "*KOREYA-가 so'z bot*'iga xush kelibsiz!\n"
        f"Bu yerda {len(CHAPTERS)} ta mavzu va {len(ALL_WORDS)} ta so'z bor.\n\n"
        "Admin: @whsdvv\n\n"
        "Quyidagilardan birini tanlang:"
    )
    await update.message.reply_text(text, reply_markup=main_menu_kb(), parse_mode=ParseMode.MARKDOWN)


# ---------------------------------------------------------------
# 5) TUGMA BOSILGANDA ISHLAYDIGAN FUNKSIYA
# ---------------------------------------------------------------

async def button_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    data = query.data

    if data == "menu:main":
        context.user_data["awaiting_search"] = False
        text = "Quyidagilardan birini tanlang:"
        await safe_edit(query, text, main_menu_kb())
        return

    if data == "menu:chapters":
        await safe_edit(query, "📚 Mavzuni tanlang:", chapters_list_kb())
        return

    if data.startswith("chapter:"):
        _, ch_idx, w_idx = data.split(":")
        ch_idx, w_idx = int(ch_idx), int(w_idx)
        await safe_edit(query, word_card_text(ch_idx, w_idx), word_card_kb(ch_idx, w_idx))
        return

    if data == "flash:start":
        order = list(range(len(ALL_WORDS)))
        random.shuffle(order)
        context.user_data["flash_order"] = order
        context.user_data["flash_pos"] = 0
        await show_flash_front(query, context)
        return

    if data == "flash:next":
        order = context.user_data.get("flash_order")
        if not order:
            order = list(range(len(ALL_WORDS)))
            random.shuffle(order)
            context.user_data["flash_order"] = order
            context.user_data["flash_pos"] = 0
        else:
            context.user_data["flash_pos"] = (context.user_data.get("flash_pos", 0) + 1) % len(order)
        await show_flash_front(query, context)
        return

    if data == "flash:shuffle":
        order = list(range(len(ALL_WORDS)))
        random.shuffle(order)
        context.user_data["flash_order"] = order
        context.user_data["flash_pos"] = 0
        await show_flash_front(query, context)
        return

    if data == "flash:flip":
        await show_flash_back(query, context)
        return

    if data == "menu:search":
        context.user_data["awaiting_search"] = True
        text = "🔍 Qidirmoqchi bo'lgan so'zni (koreyscha yoki o'zbekcha) yozib yuboring:"
        await safe_edit(query, text, InlineKeyboardMarkup(
            [[InlineKeyboardButton("🏠 Bosh menyu", callback_data="menu:main")]]
        ))
        return


async def show_flash_front(query, context):
    order = context.user_data["flash_order"]
    pos = context.user_data["flash_pos"]
    w = ALL_WORDS[order[pos]]
    p = POS_EMOJI.get(w["posClass"], w["pos"])
    text = (
        f"🎴 Flashcard  ({pos+1}/{len(order)})\n\n"
        f"🇰🇷 *{w['kr']}*  ({p})\n\n"
        f"_Javobni ko'rish uchun pastdagi tugmani bosing_ 👇"
    )
    await safe_edit(query, text, flash_front_kb())


async def show_flash_back(query, context):
    order = context.user_data["flash_order"]
    pos = context.user_data["flash_pos"]
    w = ALL_WORDS[order[pos]]
    p = POS_EMOJI.get(w["posClass"], w["pos"])
    text = (
        f"🎴 Flashcard  ({pos+1}/{len(order)})\n\n"
        f"🇰🇷 *{w['kr']}*  ({p})\n"
        f"🇺🇿 {w['uz']}\n\n"
        f"_Misol:_\n{w['exKr']}\n{w['exUz']}"
    )
    await safe_edit(query, text, flash_back_kb())


# ---------------------------------------------------------------
# 6) ODDIY MATN XABAR KELGANDA (Qidiruv uchun)
# ---------------------------------------------------------------

async def text_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not context.user_data.get("awaiting_search"):
        await update.message.reply_text(
            "Iltimos, tugmalardan foydalaning 👇 yoki /start bosing.",
            reply_markup=main_menu_kb(),
        )
        return

    query_text = update.message.text.strip().lower()
    results = [
        w for w in ALL_WORDS
        if query_text in w["kr"].lower()
        or query_text in w["uz"].lower()
        or query_text in w["exUz"].lower()
    ]

    if not results:
        await update.message.reply_text(
            f"❌ \"{update.message.text}\" bo'yicha hech narsa topilmadi. Boshqa so'z bilan urinib ko'ring.",
        )
        return

    results = results[:10]
    for w in results:
        p = POS_EMOJI.get(w["posClass"], w["pos"])
        text = (
            f"*{w['chapter_title']}*\n\n"
            f"🇰🇷 *{w['kr']}*  ({p})\n"
            f"🇺🇿 {w['uz']}\n\n"
            f"_Misol:_\n{w['exKr']}\n{w['exUz']}"
        )
        try:
            await update.message.reply_text(text, parse_mode=ParseMode.MARKDOWN)
        except Exception:
            await update.message.reply_text(text)

    context.user_data["awaiting_search"] = False
    await update.message.reply_text("Yana qidirish uchun 🔍 tugmasini bosing:", reply_markup=main_menu_kb())


# ---------------------------------------------------------------
# 7) BOTNI ISHGA TUSHIRISH
# ---------------------------------------------------------------

def main():
    if BOT_TOKEN == "BU_YERGA_TOKENINGIZNI_QOYING":
        raise SystemExit(
            "❌ BOT_TOKEN topilmadi! README.md faylidagi 1-bosqichni bajaring."
        )

    app = Application.builder().token(BOT_TOKEN).build()

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CallbackQueryHandler(button_handler))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, text_handler))

    logger.info("Bot ishga tushdi...")
    app.run_polling()


if __name__ == "__main__":
    main()# 🇰🇷 Koreya-가 so'z botini yaratish — QADAM-BAQADAM

Salom! Keling, sizning veb-saytingizni Telegram botga aylantiramiz.
Buni LEGO yig'ish kabi tasavvur qiling: bizda allaqachon barcha
"qismlar" (kod, so'zlar) tayyor. Sizga faqat ularni bir-biriga
ulash kerak. 4 ta katta qadam bor:

```
1-QADAM: Botga ism va "parol" (token) olish   (5 daqiqa)
2-QADAM: Faylларни tayyorlash                 (allaqachon tayyor!)
3-QADAM: Serverga (hosting) joylash           (10 daqiqa)
4-QADAM: Botni sinab ko'rish                  (1 daqiqa)
```

---

## 📦 Sizga berilgan fayllar

| Fayl | Bu nima |
|---|---|
| `bot.py` | Botning "miyasi" — barcha mantiq shu yerda |
| `words.json` | Saytingizdagi barcha 156 ta so'z (12 mavzu) |
| `requirements.txt` | Botga kerakli "qo'shimcha dasturlar" ro'yxati |
| `Procfile` | Serverga "botni qanday ishga tushirish kerak" deb aytadigan fayl |

Hech narsa o'zgartirishingiz shart emas — hammasi ishlashga tayyor.

---

## 1-QADAM: Telegramdan bot oling

Telegramda **BotFather** degan maxsus bot bor — u boshqa botlarni
"tug'diradi". Xuddi pasport beruvchi idoraga o'xshaydi.

1. Telegram'da qidiruvga **@BotFather** deb yozing va oching.
2. **/newbot** buyrug'ini yuboring.
3. Botingizga ism bering, masalan: `Koreya Words Bot`
4. Botingizga **username** bering — bu albatta `bot` bilan tugashi kerak,
   masalan: `koreya_words_bot` (agar band bo'lsa, boshqa nom sinab ko'ring).
5. BotFather sizga shunday narsa yuboradi:

   ```
   123456789:AAFf3kLmN9opQrStUvWxYz1234567890abc
   ```

   **Mana shu — sizning bot tokeningiz.** Bu botingizning paroli,
   uni hech kimga bermang, screenshot olmang, chatga yozmang.
   Faylga yoki yopiq joyga saqlab qo'ying.

---

## 2-QADAM: Fayllarni tekshirish

Quyida sizga 4 ta fayl tayyorladim. Ularni kompyuteringizga yuklab
oling va bitta papkaga joylang, masalan: `koreya-bot/`

```
koreya-bot/
 ├─ bot.py
 ├─ words.json
 ├─ requirements.txt
 └─ Procfile
```

---

## 3-QADAM: Serverga (hosting) joylash

Botingiz 24 soat ishlab turishi uchun u **doim yoniq turadigan
kompyuterda** (server) ishlashi kerak — sizning shaxsiy
kompyuteringiz emas (chunki uni o'chirib qo'yasiz, bot ham o'chadi).

Eng oson va boshlovchilar uchun qulay xizmat — **Railway.app**
(bepul tarif bor). Quyida shu orqali tushuntiraman:

### a) Railway'da hisob oching
1. https://railway.app saytiga kiring
2. **"Login with GitHub"** orqali ro'yxatdan o'ting (GitHub akkaunt
   kerak bo'ladi — bo'lmasa https://github.com da bepul oching)

### b) Kodni GitHub'ga yuklang
1. https://github.com da **"New repository"** tugmasini bosing
2. Nom bering, masalan: `koreya-bot`, **Create** bosing
3. O'sha repo sahifasida **"uploading an existing file"** havolasini
   bosing va 4 ta faylingizni (`bot.py`, `words.json`,
   `requirements.txt`, `Procfile`) sudrab tashlang (drag & drop)
4. Pastda **"Commit changes"** tugmasini bosing

### c) Railway'ni GitHub'ga ulang
1. Railway'da **"New Project"** → **"Deploy from GitHub repo"**
2. Yaratgan `koreya-bot` repongizni tanlang
3. Railway avtomatik aniqlaydi va qurishni (build) boshlaydi

### d) Tokenni Railway'ga kiriting (eng muhim qadam!)
1. Railway loyihangiz ichida **"Variables"** bo'limini toping
2. Yangi o'zgaruvchi qo'shing:
   - **Name:** `BOT_TOKEN`
   - **Value:** BotFather bergan o'sha uzun kod
     (`123456789:AAFf3kL...`)
3. Saqlang — Railway botni avtomatik qayta ishga tushiradi

✅ Tayyor! Endi botingiz internetda, 24/7 ishlab turibdi.

> 💡 Agar Railway o'rniga boshqa xizmat (Render.com, PythonAnywhere,
> yoki o'z VPS serveringiz) ishlatmoqchi bo'lsangiz — printsip bir xil:
> fayllarni yuklaysiz, `BOT_TOKEN` degan environment variable
> qo'shasiz, va `python bot.py` buyrug'i bilan ishga tushirasiz.

---

## 4-QADAM: Botni sinab ko'ring

1. Telegram'da o'zingiz yaratgan bot username'ini qidiring
   (masalan `@koreya_words_bot`)
2. **/start** bosing
3. Tugmalarni bosib ko'ring:
   - **📚 Mavzular** → 12 ta mavzudan birini tanlang → so'zlar orasida
     "⬅️ Oldingi" / "Keyingi ➡️" bilan yuring
   - **🎴 Flashcard** → koreyscha so'z chiqadi → "🔄 Javobni ko'rsatish"
     bosib tarjimasini ko'ring
   - **🔍 Qidirish** → istalgan so'zni (koreyscha yoki o'zbekcha) yozib
     yuboring, bot mos so'zlarni topib beradi

---

## 🖥️ (Ixtiyoriy) O'z kompyuteringizda sinab ko'rish

Serverga qo'yishdan oldin shaxsiy kompyuteringizda tekshirib
ko'rmoqchi bo'lsangiz:

```bash
# 1) Python o'rnatilganini tekshiring (terminalda yozing):
python3 --version

# 2) Papkaga kiring:
cd koreya-bot

# 3) Kerakli kutubxonani o'rnating:
pip install -r requirements.txt

# 4) Tokenni vaqtincha kompyuteringizga kiriting:
export BOT_TOKEN="BotFather_bergan_token_shu_yerga"      # Mac/Linux
set BOT_TOKEN=BotFather_bergan_token_shu_yerga            # Windows (cmd)

# 5) Botni ishga tushiring:
python3 bot.py
```

Terminalda `Bot ishga tushdi...` deb chiqsa — Telegram'da botingizga
**/start** yozib sinab ko'rishingiz mumkin (kompyuteringiz ochiq
turgunicha bot ishlaydi).

---

## ❓ Tez-tez so'raladigan savollar

**"Unauthorized" degan xato chiqsa?**
Token noto'g'ri nusxalangan. BotFather'dan qaytadan oling, oldidagi
yoki ortidagi bo'shliqlarni olib tashlang.

**Bot javob bermayapti?**
Railway'da "Deployments" bo'limidan loglarni (xatoliklar yozuvi)
tekshiring — odatda u yerda sabab yozilgan bo'ladi.

**Yangi so'z qo'shmoqchiman?**
`words.json` faylini ochib, kerakli mavzu ichiga shu shaklda
yangi qator qo'shing:
```json
{"kr": "안녕", "pos": "Ot", "posClass": "noun", "uz": "salom",
 "exKr": "*안녕*하세요", "exUz": "Salom (rasmiy)"}
```
So'ng faylni GitHub'ga qayta yuklasangiz, Railway avtomatik yangilaydi.

---

Savol tug'ilsa — shu yerda yozing, men har bir qadamni birga
bosib o'tamiz 🙂
