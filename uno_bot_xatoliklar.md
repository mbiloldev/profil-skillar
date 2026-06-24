# 🔍 UNO Bot — To'liq Tekshiruv Hisoboti

Skanerlangan fayllar: `bot.py`, `config.py`, `deck.py`, `game.py`, `keyboards.py`, `stickers.py`, `handlers/lobby.py`, `handlers/gameplay.py`, `handlers/misc.py`

---

## 🔴 KRITIK XATOLAR (Crash / Noto'g'ri mantiq)

---

### BUG 1 — `game.py`: `leave_game` → `IndexError` (Crash!)

**Fayl:** `game.py` → `GameState.current_player`  
**Holat:** O'yin boshlangan, 3 yoki undan ko'p o'yinchi bor, `current_index = 2`, keyin **3-o'yinchi chiqib ketadi**.  

```python
# leave_game da players dan o'chiriladi, lekin current_index kamaymaydi
# game.players = [P0, P1]  lekin  current_index = 2  => IndexError!
@property
def current_player(self) -> Player:
    return self.players[self.current_index]  # ← IndexError!
```

**Muammo:** `remove_player` `current_index` ni tekshirmaydi.

**To'g'irlash:**
```python
def remove_player(self, user_id: int) -> Player | None:
    player = self.get_player(user_id)
    if player:
        idx = self.players.index(player)
        self.players.remove(player)
        # current_index ni tuzatish
        if self.started and self.players:
            if idx < self.current_index:
                self.current_index -= 1
            self.current_index = self.current_index % len(self.players)
    return player
```

---

### BUG 2 — `deck.py`: `can_be_played_on` → `None` qaytaradi (Boolean o'rniga)

**Fayl:** `deck.py` → `Card.can_be_played_on`  
**Holat:** `stack_type = '+2'`, `active_color = None`, `self.value = '+2'`, ranglar mos kelmaydi.

```python
if stack_type == "+2":
    if self.value == "+2":
        return self.color == top.color or (active_color and self.color == active_color)
        #       False          or         (None and ...)
        #       False          or         None
        #       => None  ← Boolean emas!
```

`False or None` → Python `None` qaytaradi. Bu `playable_cards()` da Falsy bo'ladi, ya'ni o'yinchi o'z +2 kartasini stack qila olmaydi (bloklangan ko'rinadi) — lekin hech qachay aniq `False` emas.

**To'g'irlash:**
```python
return bool(self.color == top.color or (active_color and self.color == active_color))
```

---

### BUG 3 — `handlers/gameplay.py`: `announce_winner` → `KeyError` ehtimoli

**Fayl:** `handlers/gameplay.py` → `announce_winner`  
**Holat:** Funksiya ichida `del G.games[g.game_id]` so'zsiz chaqiriladi. Agar o'yin allaqachon o'chirilgan bo'lsa (masalan, `leave_game` orqali) — `KeyError` crashga olib keladi.

```python
# Hozirgi kod:
del G.games[g.game_id]  # ← guard yo'q

# To'g'irlash:
G.games.pop(g.game_id, None)
```

---

## 🟠 O'RTA DARAJALI XATOLAR (Noto'g'ri xulq-atvor)

---

### BUG 4 — `deck.py`: `build_deck()` → 118 karta (README: 114)

**Fayl:** `deck.py` → `build_deck()`

Hisob:
| Karta turi | Soni |
|---|---|
| Raqamli (har rang: 0×1 + 1-9×2 = 19) × 4 rang | 76 |
| Action (skip, reverse, +2 × 2) × 4 rang | 24 |
| Wild | 6 |
| +4 Wild | 8 |
| +10 Wild | 4 |
| **Jami** | **118** |

README `114` deydi. Agar 114 bo'lishi kerak bo'lsa: Wild ni 6→4, +4 ni 8→4 qilish kerak (standart UNO). Aks holda README ni 118 ga yangilash kerak.

---

### BUG 5 — `handlers/lobby.py`: Typo (matn xatosi)

**Fayl:** `handlers/lobby.py`, satr ~80

```python
await message.answer("Siz allaqachon bu o'yindasi z.")
#                                          ^^^^^^^^ typo!
# To'g'risi:
await message.answer("Siz allaqachon bu o'yindasiz.")
```

---

### BUG 6 — `game.py`: `play_card` → `+2` stacking paytida `stack_type` mantiq

**Fayl:** `game.py` → `play_card`

```python
elif card.value in ("+2", "+4", "+10"):
    self.pending_draw += card.draw_value
    if card.value in ("+4", "+10"):
        self.stack_type = card.value
    else:
        self.stack_type = self.stack_type or "+2"
```

**Muammo:** Agar `stack_type` allaqachon `'+4'` bo'lsa va `+2` o'ynansa (bu aslida `can_be_played_on` tomonidan bloklanishi kerak), `stack_type` `'+4'` bo'lib qoladi. Mantiqan to'g'ri, lekin `pending_draw` noto'g'ri oshishi mumkin. Bu holat `can_be_played_on` da bloklanadi, lekin qo'shimcha guard bo'lishi yaxshi.

---

## 🟡 KICHIK XATOLAR (Code quality)

---

### BUG 7 — `handlers/misc.py`: Keraksiz import

```python
from aiogram.filters import Command, CommandStart
#                                    ^^^^^^^^^^^^ ishlatilmaydi
```
`CommandStart` import qilingan, lekin `misc.py` da hech qayerda handler yo'q. Olib tashlash kerak.

---

### BUG 8 — `keyboards.py`: `join_keyboard` — Dead Code

`join_keyboard(game_id)` funksiyasi yozilgan, lekin hech qayerda chaqirilmaydi. Yoki ishlatilishi kerak (masalan, guruh chatda o'yin e'lon qilinganda), yoki o'chirilishi kerak.

---

### BUG 9 — `handlers/lobby.py`: `cmd_start_plain` va `cmd_start_deep` tartib muammosi

**Fayl:** `handlers/lobby.py`

```python
@router.message(CommandStart())          # ← oddiy /start
@router.message(CommandStart(deep_link=True))  # ← /start join_xxx
```

aiogram 3.x da `CommandStart(deep_link=True)` ni **oldin** ro'yxatga olish kerak, aks holda oddiy `CommandStart()` handler ham deep link uchun ishlab, payload e'tibordan chetda qolishi mumkin. Router tartibi muhim.

**To'g'irlash:** `cmd_start_deep` ni `cmd_start_plain` dan **oldin** yozing (faylda tartibni almashtiring).

---

## ✅ TO'G'RI ISHLAYDIGAN FUNKSIYALAR

| Funksiya | Holat |
|---|---|
| `build_deck()` — karta tuzilishi | ✅ (faqat son nomuvofiq) |
| `Card.can_be_played_on` — asosiy holatlar | ✅ |
| `Card.draw_value`, `is_wild`, `is_draw_card` | ✅ |
| `Card.label`, `sticker_key` | ✅ |
| `Deck.draw()` / `draw_many()` | ✅ |
| `Deck._reshuffle()` | ✅ |
| `Deck.flip_start_card()` — wild qaytarmaydi | ✅ |
| `GameState.start_game()` | ✅ |
| `GameState.advance_turn()` — modulo wrap | ✅ |
| `GameState.advance_turn(skip=True)` | ✅ |
| `GameState.reverse_direction()` | ✅ |
| `GameState.play_card()` — asosiy oqim | ✅ |
| `GameState.play_card()` — g'olib aniqlash | ✅ |
| `GameState.play_card()` — 2 kishi reverse=skip | ✅ |
| `GameState.pick_color()` | ✅ |
| `GameState.draw_card()` — oddiy | ✅ |
| `GameState.draw_card()` — forced (pending_draw) | ✅ |
| `GameState.playable_cards()` | ✅ |
| `GameState.status_text()` / `scores_text()` | ✅ |
| `create_game()` / `join_game()` / `leave_game()` | ✅ (BUG 1 dan tashqari) |
| `lobby_keyboard()` / `hand_keyboard()` / `color_keyboard()` | ✅ |
| `get_sticker()` — placeholder himoyasi | ✅ |
| `broadcast_lobby()` — edit yoki yangi xabar | ✅ |
| `broadcast_status()` — edit yoki yangi xabar | ✅ |
| `send_hand_to_player()` | ✅ |
| `cb_play_card()` — UNO e'loni | ✅ |
| `cb_draw_card()` | ✅ |
| `cb_pick_color()` | ✅ |
| `cmd_quit()` — qolgan o'yinchilarga xabar | ✅ |
| `cmd_cards()` — kartalarni qayta yuborish | ✅ |
| `cmd_status()` | ✅ |
| `bot.py` — router tartibi | ✅ |

---

## 📋 Tuzatish Prioriteti

| # | Bug | Og'irlik | Fayl |
|---|---|---|---|
| 1 | `leave_game` → `IndexError` crash | 🔴 Kritik | `game.py` |
| 2 | `can_be_played_on` → `None` qaytaradi | 🔴 Kritik | `deck.py` |
| 3 | `announce_winner` → `KeyError` | 🔴 Kritik | `gameplay.py` |
| 4 | Deck karta soni mismatch (118 vs 114) | 🟠 O'rta | `deck.py` + README |
| 5 | `"bu o'yindasi z"` typo | 🟡 Kichik | `lobby.py` |
| 6 | `cmd_start_deep` tartib muammosi | 🟡 Kichik | `lobby.py` |
| 7 | `CommandStart` keraksiz import | 🟡 Kichik | `misc.py` |
| 8 | `join_keyboard` dead code | 🟡 Kichik | `keyboards.py` |
