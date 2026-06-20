# 🇰🇷 Koreya-가 so'z botini yaratish — QADAM-BAQADAM

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
