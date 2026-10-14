# ALGO MIND — สเปกระบบเทรด v1.0 (กฎแบบ A)

> สถานะ: **ร่างสำหรับ backtest** — ทุกค่าเป็นค่าเริ่มต้นที่ต้องยืนยันด้วยข้อมูลจริงก่อนใช้เงินจริง
> ใช้ได้กับ: Forex (Major pairs), XAUUSD, BTCUSD | สไตล์: Day Trade และ Swing Trade

---

## 0. หลักการของระบบ

ระบบตอบ 3 คำถามตามลำดับ แล้วผ่านด่านความเสี่ยงก่อนเข้าเทรด

| # | คำถาม | โมดูล | สถานะ |
|---|---|---|---|
| ① | ทิศไหน? | Market Structure (TF ใหญ่) | **บังคับ** |
| ③ | ที่ไหน? | Volume Profile (โซนเข้า) | **บังคับ** |
| ⑤ | เมื่อไหร่? | Price Action (TF เล็ก) | **บังคับ** |
| ② | โมเมนตัมเห็นด้วยไหม? | MACD (TF เล็ก) | เสริม → กำหนดเกรด |
| ⑥ | ออกที่ไหน? | VP levels + Fib Extension | ใช้วาง TP เท่านั้น |
| ④ | คุ้มและปลอดภัยไหม? | Risk Gate | **บังคับ** |

**กฎแบบ A:** ①+③+⑤+④ ผ่าน → เข้าเทรดได้
- มี ② ด้วย → **เกรด A** (เสี่ยง 1.0%)
- ไม่มี ② → **เกรด B** (เสี่ยง 0.5%)

---

## 1. Timeframe

| บทบาท | Day Trade | Swing Trade |
|---|---|---|
| Bias TF (โมดูล ①) | H4 | D1 |
| Setup TF (โซน ③, Fib ⑥) | H1 | H4 |
| Entry TF (โมดูล ⑤ ②) | M30 | H1 |
| Volume Profile | วันก่อนหน้า (Prior Day) | Composite 20 วัน |
| Time stop | ปิดก่อนจบ NY session | สูงสุด 10 วันทำการ |

พารามิเตอร์: `TradeStyle = DAY | SWING`

---

## 2. โมดูล ① Market Structure (บังคับ)

**การหาจุดสวิง (Fractal)**
- Swing High = แท่งที่ High สูงกว่า `SwingLen` แท่งทั้งซ้ายและขวา (Swing Low กลับด้าน)
- ไม่นับสวิงที่ห่างจากสวิงก่อนหน้าน้อยกว่า `MinSwingATR × ATR(14)` เพราะถือเป็นสัญญาณรบกวน

**การตัดสิน Bias**
- **BOS ขึ้น** = แท่งปิดเหนือ Swing High ล่าสุด
- **BOS ลง** = แท่งปิดต่ำกว่า Swing Low ล่าสุด
- `BULL` = เหตุการณ์ล่าสุดคือ BOS ขึ้น และยังไม่มีแท่งปิดต่ำกว่า HL ล่าสุด
- `BEAR` = กลับด้านของ BULL
- กรณีอื่น = `NEUTRAL` → **ไม่เทรด**

| พารามิเตอร์ | ค่าเริ่มต้น |
|---|---|
| `SwingLen` | Day = 3, Swing = 5 |
| `MinSwingATR` | 0.5 |

---

## 3. โมดูล ③ Volume Profile (บังคับ)

**การคำนวณ**
- แบ่งช่วงราคาเป็น `VP_Bins` ช่อง แล้วกระจาย volume ของแท่ง M1/M5 ลงในแต่ละช่อง
- **POC** = ช่องที่มี volume สูงสุด
- **VAH/VAL** = ขอบบนและล่างของช่วงที่มี volume รวม `VA_Percent` รอบ POC
- **HVN** = จุดยอด volume ในพื้นที่ที่สูงกว่า `HVN_Mult × ค่าเฉลี่ย` (เก็บไม่เกิน 3 จุดที่แรงที่สุด)
- **LVN** = จุดต่ำ volume ในพื้นที่ ใช้บอกว่าราคามักวิ่งผ่านเร็ว (ไม่ใช้เป็นโซนเข้า)

**แหล่ง volume ตามสินทรัพย์**
- Forex = tick volume ของ broker
- XAUUSD = tick volume (ถ้าดึง GC futures ได้ให้ใช้ GC แทน)
- BTCUSD = volume จริงจาก exchange

**โซนเข้า**
- **Buy:** VAL, POC หรือ HVN ที่อยู่**ใต้**ราคา
- **Sell:** VAH, POC หรือ HVN ที่อยู่**เหนือ**ราคา
- ความกว้างโซน = level ± `ZoneWidthATR × ATR(14, Setup TF)`
- **เงื่อนไขผ่าน:** Low (สำหรับ Buy) หรือ High (สำหรับ Sell) แตะโซนภายใน `ZoneTouchBars` แท่งล่าสุดของ Entry TF และราคาปิดยังไม่ทะลุโซนออกไปเกิน 1 ATR

| พารามิเตอร์ | ค่าเริ่มต้น |
|---|---|
| `VP_Bins` | 100 |
| `VA_Percent` | 70% |
| `HVN_Mult` | 1.5 |
| `ZoneWidthATR` | 0.25 |
| `ZoneTouchBars` | 6 |

---

## 4. โมดูล ⑤ Price Action Trigger (บังคับ)

ใช้ Entry TF และต้องเกิดภายในโซนหรือภายใน `ZoneTouchBars` หลังแตะโซน ผ่านข้อใดข้อหนึ่งก็นับ (ฝั่ง Buy อธิบายด้านล่าง ฝั่ง Sell กลับด้าน)

1. **CHoCH บน TF เล็ก:** หาสวิงด้วย `LTF_SwingLen` ราคาทำ Low ใหม่ในโซน แล้วมีแท่ง**ปิด**เหนือ Lower High ล่าสุด
2. **Bullish Engulfing:**
   - แท่งก่อนหน้าเป็นแท่งแดง
   - แท่งปัจจุบันเป็นแท่งเขียว
   - Close > Open ของแท่งก่อนหน้า และ Open ≤ Close ของแท่งก่อนหน้า
   - ขนาดตัวแท่ง ≥ `EngulfMinBodyATR × ATR`

**การเข้า:** Market order ตอนปิดแท่ง trigger (v1) ส่วนการตั้ง limit รอ retest เก็บไว้ทดสอบเป็นตัวเลือกใน v2

| พารามิเตอร์ | ค่าเริ่มต้น |
|---|---|
| `LTF_SwingLen` | 2 |
| `EngulfMinBodyATR` | 0.5 |
| `EnableCHoCH` / `EnableEngulf` | true / true |

---

## 5. โมดูล ② MACD (เสริม → เกรด A)

ใช้ MACD(12, 26, 9) บน Entry TF ผ่านข้อใดข้อหนึ่งก็นับ (ฝั่ง Buy อธิบายด้านล่าง)

1. **Histogram กลับทิศ:** ที่แท่ง trigger หรือ 2 แท่งก่อนหน้า histogram เปลี่ยนจากลบเป็นบวก หรือติดลบแต่สูงขึ้นติดกัน 2 แท่ง
2. **Bullish Divergence:** ภายใน `DivLookback` แท่ง ราคาทำ Lower Low แต่ MACD line ทำ Higher Low และ Low ตัวหลังอยู่ในโซน VP

| พารามิเตอร์ | ค่าเริ่มต้น |
|---|---|
| `MACD_Fast / Slow / Signal` | 12 / 26 / 9 |
| `DivLookback` | 20 |
| `UseMACDGrade` | true (ถ้าปิด = ทุกไม้เป็นเกรด B สำหรับ backtest เปรียบเทียบ) |

---

## 6. จุดออก (SL / TP)

### Stop Loss
- **Buy:** SL = min(ขอบล่างของโซน, Low ของ trigger swing) − `SLBufferATR × ATR(14, Entry TF)`
- **Sell:** SL = max(ขอบบนของโซน, High ของ trigger swing) + buffer
- ถ้าระยะ SL > `MaxSL_ATR × ATR(14, Setup TF)` → **ข้ามเทรด** เพราะ SL กว้างเกินไป

### Fibonacci Extension (Buy, ฝั่ง Sell กลับด้าน)
- A = Swing Low ต้นขาขึ้นใน Setup TF
- B = Swing High ที่ทำ BOS
- C = Low ต่ำสุดตอนแตะโซน
- `Ext(r) = C + (B − A) × r` โดย r ∈ {1.0, 1.272, 1.618} **ไม่ใช้ระดับอื่น**

### Take Profit

| เป้า | วิธีเลือก | ปิด |
|---|---|---|
| TP1 | VP level แรกข้างหน้า (POC/VAH/HVN) ที่ RR ≥ 1.0 | 50% + เลื่อน SL ไป BE |
| TP2 | VP level ถัดไป ถ้าไม่มีให้ใช้ Fib 1.272 | 30% |
| TP3 | Fib 1.618 | 20% ที่เหลือ ใช้ trailing stop |

**ลำดับความสำคัญ**
- ถ้า VP level กับ Fib ext ห่างกันไม่เกิน `ConfluenceATR` → ใช้ระดับนั้นเป็น **เป้าแข็งแรง** และปิดสถานะส่วนที่เหลือทั้งหมดที่ระดับนี้
- ถ้า TP1 กับ TP2 ห่างกันไม่ถึง 0.5R → รวมเป็น TP เดียว

**Trailing:** หลังถึง TP2 ให้เลื่อน SL ตาม Swing Low ของ Entry TF

**Time stop**
- Day trade: ปิดทั้งหมดที่ `DayCloseTime`
- Swing: ปิดเมื่อถือครบ `SwingMaxDays`

| พารามิเตอร์ | ค่าเริ่มต้น |
|---|---|
| `SLBufferATR` | 0.3 |
| `MaxSL_ATR` | 2.0 |
| `ConfluenceATR` | 0.3 |
| `TP_Split` | 50 / 30 / 20 |
| `DayCloseTime` | 22:00 เวลาเซิร์ฟเวอร์ (ปรับตาม broker) |
| `SwingMaxDays` | 10 |

---

## 7. ด่านความเสี่ยง ④ (บังคับ — ไม่ผ่านแม้ข้อเดียวก็ไม่เข้า)

**คุณภาพเทรด**
- RR (Entry → TP2) ≥ `MinRR`
- RR (Entry → Fib 1.0) ≥ `MinRR_Fib100` ถ้าไม่ถึง แปลว่าแรงไม่พอ → ข้าม

**ขนาดสถานะ**
```
RiskAmount = Balance × Risk%          (A = 1.0%, B = 0.5%)
Lot        = RiskAmount / (SL_distance_in_price × ValuePerPricePerLot)
```
ปัดลงตาม lot step ของ broker และถ้าเล็กกว่า min lot → ข้าม

**ขีดจำกัด**

| กฎ | ค่าเริ่มต้น |
|---|---|
| `MinRR` | 2.0 |
| `MinRR_Fib100` | 1.5 |
| `MaxOpenTrades` | 3 รวมทุกสินทรัพย์, 1 ต่อ symbol |
| `MaxCurrencyExposure` | ความเสี่ยงรวมต่อสกุลเงินเดียว ≤ 2% |
| `DailyLossLimit` | −3% → หยุดถึงวันถัดไป |
| `WeeklyLossLimit` | −6% → หยุดจนกว่าคนจะตรวจสอบ |
| `MaxConsecLosses` | 4 → หยุดวันนั้น |
| `NewsBlackout` | ±30 นาทีรอบข่าว High impact |
| `MaxSpreadMult` | spread > 2 × ค่ามัธยฐาน 20 วัน → ข้าม |
| `FridayCutoff` | Forex/ทอง ไม่เปิดสถานะใหม่หลัง 20:00 วันศุกร์ (เวลาเซิร์ฟเวอร์) |

---

## 8. ค่าเฉพาะสินทรัพย์

| | Forex Majors | XAUUSD | BTCUSD |
|---|---|---|---|
| เวลาเทรด | London + NY | London + NY | 24/7 |
| Volume สำหรับ VP | tick | tick / GC futures | exchange จริง |
| `SLBufferATR` | 0.3 | 0.4 | 0.5 |
| `MaxSpreadMult` | 2.0 | 2.0 | 2.5 |
| สุดสัปดาห์ | ปิดตลาด | ปิดตลาด | ลดความเสี่ยงลง 50% |
| ข้อควรระวัง | คู่ที่ correlate กัน | ข่าว USD/FOMC แรงมาก | Funding/swap สูง, ช่วง low liquidity |

---

## 9. Flow การตัดสินใจ (pseudocode)

```text
ON new bar close (Entry TF):
  if RiskGate.halted(): return                         # daily/weekly/consecutive loss
  bias = Structure.bias(BiasTF)                         # ①
  if bias == NEUTRAL: log("no bias"); return
  zone = VP.activeZone(bias, ZoneTouchBars)             # ③
  if zone is None: return
  trig = PriceAction.trigger(bias, zone)                # ⑤
  if trig is None: return
  grade = MACD.confirm(bias, zone) ? "A" : "B"          # ②
  sl   = Exits.stopLoss(bias, zone, trig)
  tps  = Exits.targets(bias, entry, sl, VP, FibExt)     # ⑥
  check = RiskGate.validate(entry, sl, tps, grade,      # ④
                            spread, news, exposure)
  log(signal, all module states, check)                 # บันทึกทุกครั้ง แม้ไม่ผ่าน
  if check.ok: Execution.place(entry, sl, tps, check.lot)
```

---

## 10. การบันทึก (Journal)

ทุกสัญญาณ **รวมถึงสัญญาณที่ถูกปฏิเสธ** ให้บันทึกข้อมูลต่อไปนี้:
- เวลา, symbol, style, bias
- โซน VP ที่ใช้ (ประเภทและราคา), ประเภท trigger, ผล MACD, เกรด
- Entry/SL/TP1–3, RR, lot
- เหตุผลที่ไม่ผ่าน (ถ้ามี)
- ผลลัพธ์: R ที่ได้, MAE/MFE, เวลาที่ถือ

ข้อมูลชุดนี้ใช้วิเคราะห์ว่าแต่ละโมดูลช่วยระบบจริงหรือไม่

---

## 11. แพลตฟอร์ม: TradingView + Eightcap (ตัดสินใจแล้ว)

**โค้ด:** `pine/algomind_xauusd_day_v1.pine` (Pine Script v6 `strategy()`) สำหรับชาร์ต `EIGHTCAP:XAUUSD` 30m

**เส้นทางส่งคำสั่ง**
```
TradingView strategy ──alert() JSON──► Webhook ──► Bridge ──► MT5 (บัญชี Eightcap)
                                                   (PineConnector หรือ backend ALGO MIND)
```
- TradingView ไม่ส่งคำสั่งของ strategy ไปที่ broker อัตโนมัติ ต้องมี bridge มารับ webhook
- Webhook ต้องใช้แพ็กเกจ TradingView แบบเสียเงิน และใน MT5 ต้องเปิด Algo Trading
- ต้องตรวจกับ Eightcap ว่าบัญชีที่ใช้อนุญาตให้รัน EA/bridge ได้ และชื่อ symbol ใน MT5 ตรงกับที่ตั้งไว้

**ข้อความ webhook ที่ script ส่งออก**
```json
{"secret":"…","symbol":"XAUUSD","action":"BUY","grade":"A","risk_pct":1.0,
 "entry":2622.10,"sl":2612.00,"tp1":2640.00,"tp2":2683.60,"tp3":2700.90,
 "split":[50,30,20],"be_after_tp1":true,"trail_after_tp2":true,
 "zone":"VAL","trigger":"CHoCH"}
{"secret":"…","symbol":"XAUUSD","action":"MODIFY_SL","sl":2622.10}
{"secret":"…","symbol":"XAUUSD","action":"CLOSE"}
```
Bridge คำนวณ lot จาก `risk_pct` กับ equity จริงของบัญชีเอง

**สิ่งที่ Pine ทำไม่ได้ ต้องทำที่ bridge/backend**
- กรองข่าว (Pine อ่านปฏิทินเศรษฐกิจไม่ได้)
- กรอง spread แบบ real-time
- ความเสี่ยงรวมข้ามหลาย symbol

**ข้อจำกัดของ backtest บน TradingView**
- ไม่มี spread จริง ใช้ `slippage` (ค่าเริ่มต้น 15 ticks) เป็นตัวแทน
- Volume Profile ใช้ tick volume ของ Eightcap และ intrabar 5m ซึ่งย้อนหลังได้จำกัดตามแพ็กเกจ
- ทุกชั่วโมงในสเปกเป็นเวลา **UTC**: session 07–20, time stop 20:00, Friday cutoff 18:00

## 12. แผนทดสอบก่อนใช้เงินจริง

1. **Backtest** ≥ 2 ปีต่อสินทรัพย์ คิด spread, commission และ slippage จริง
2. **เกณฑ์ผ่าน (เบื้องต้น)**
   - ≥ 100 เทรด
   - Profit Factor ≥ 1.3
   - Expectancy > 0.2R
   - Max Drawdown ≤ 15%
3. **เปรียบเทียบตัวแปร**
   - MACD เปิด vs ปิด
   - VP lookback (prior day vs composite)
   - CHoCH vs Engulfing
4. **Walk-forward:** ปรับค่าด้วยข้อมูล 70% แรก แล้วทดสอบกับ 30% หลัง ห้ามปรับค่าจนพอดีกับข้อมูลทั้งหมด
5. **Demo** 1–2 เดือน แล้วเทียบผลกับ backtest
6. **Live เล็ก** เสี่ยง 0.25% ต่อไม้จนครบ 50 เทรด แล้วค่อยใช้ค่าตามสเปก

---

## 13. การตัดสินใจ

- [x] แพลตฟอร์ม: **TradingView**
- [x] Broker: **Eightcap** (ส่งคำสั่งผ่าน MT5)
- [x] เริ่มที่: **XAUUSD + Day Trade**
- [ ] Bridge: PineConnector (สำเร็จรูป) หรือ backend ALGO MIND (สร้างเอง)
