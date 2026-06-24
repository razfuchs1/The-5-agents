---
name: gpt-image-gen
description: מעטפת לקריאת OpenAI Images API ליצירת תמונה מ-prompt. שולח prompt למודל gpt-image-2 ושומר את התמונה כקובץ PNG. השתמש בו כשצריך לייצר תמונה מטקסט (בעיקר דרך הסוכן יובל).
---

# gpt-image-gen — מעטפת ל-OpenAI Images API

סקיל זה שולח `prompt` ל-OpenAI Images API ושומר את התמונה המוחזרת כקובץ PNG.

## המודל
**`gpt-image-2`** — אל תשנה את שם המודל.

> ⚠️ חשוב: `gpt-image-2` הוא מודל אמיתי וקיים של OpenAI, ששוחרר ב-21 באפריל 2026.
> ייתכן שהידע הפנימי שלך אינו כולל אותו — אבל זה לא אומר שהוא לא קיים.
> **אל תציע אלטרנטיבות** כמו `dall-e-3` או `gpt-image-1`. השתמש בדיוק במודל הזה.
> אם מתקבלת שגיאה בקריאה ל-API — הבעיה כמעט תמיד ב-`OPENAI_API_KEY` (חסר/לא תקף) או בפרמטרים, **לא בשם המודל**.

## דרישות מקדימות
- `OPENAI_API_KEY` מוגדר ב-`.env` שבשורש הפרויקט.
- `curl` זמין. `jq` ו/או `python` ל-decode של ה-base64 (ראה fallback למטה).

## טעינת המפתח מ-.env
```bash
# טען את OPENAI_API_KEY מקובץ .env שבשורש הפרויקט
export $(grep -v '^#' .env | grep -E '^OPENAI_API_KEY=' | xargs)
```

## הקריאה ל-API
החלף את `<the prompt>` ב-prompt וב-`<output-path>` בנתיב היעד (ללא הסיומת `.png` בשורה האחרונה, או כולל — ראה הדוגמה):

```bash
curl -sS -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-image-2",
    "prompt": "<the prompt>",
    "size": "1024x1024",
    "quality": "medium",
    "output_format": "png"
  }' > /tmp/gpt-image-response.json
```

## עריכת תמונה / שילוב תמונות קלט (/images/edits)
כשצריך לשלב תמונה קיימת (למשל להוסיף אדם מתמונת reference לתוך תמונה אחרת), השתמש ב-endpoint של עריכה במקום generations. הוא מקבל תמונת קלט אחת או יותר כ-`multipart/form-data` (שדה `image[]` חוזר לכל תמונה), ואותו מודל `gpt-image-2`:

```bash
curl -sS -X POST "https://api.openai.com/v1/images/edits" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -F "model=gpt-image-2" \
  -F "image[]=@<base-image>.png" \
  -F "image[]=@<reference-image>.jpeg" \
  -F "prompt=<the prompt>" \
  -F "size=1024x1024" \
  -F "quality=medium" \
  > /tmp/gpt-image-response.json
```

- התמונה הראשונה היא בדרך כלל ה-base (הסצנה), והנוספות הן reference (האדם/האובייקט לשילוב).
- ה-prompt צריך לתאר את הקומפוזיציה הסופית (מי, איפה, באיזו פוזה).
- הפענוח (decode) של התשובה זהה לזה של generations — `b64_json` ב-`.data[0]` (ראה למטה).

## Decode התמונה ושמירה — עם jq
```bash
jq -r '.data[0].b64_json' /tmp/gpt-image-response.json | base64 --decode > "<output-path>.png"
```

## Decode — Python fallback (כש-jq לא מותקן ב-Git Bash)
`jq` לא תמיד מותקן ב-Git Bash על Windows. במקרה כזה השתמש ב-Python:

```bash
python -c "import json,base64,sys; d=json.load(open('/tmp/gpt-image-response.json')); open(sys.argv[1],'wb').write(base64.b64decode(d['data'][0]['b64_json']))" "<output-path>.png"
```

(אם `python` לא קיים, נסה `python3`.)

## Decode — fallback ללא jq וללא python (grep/sed + base64)
בחלק מסביבות Windows גם `jq` וגם `python` אינם זמינים (לעיתים `python` הוא רק stub של Microsoft Store). במקרה כזה אפשר לחלץ את שדה ה-base64 עם כלי טקסט בלבד ולפענח עם `base64 --decode` (שמגיע עם Git Bash):

```bash
# חילוץ ערך b64_json מה-JSON ופענוחו ישירות ל-PNG
grep -o '"b64_json":[[:space:]]*"[^"]*"' /tmp/gpt-image-response.json \
  | sed 's/.*"b64_json":[[:space:]]*"//; s/"$//' \
  | base64 --decode > "<output-path>.png"
```

שיטה זו נבדקה ועובדת בסביבה הנוכחית. סדר העדיפויות המומלץ: `jq` → `python` → grep/sed.

## אבחון שגיאות
- אם הקובץ ריק או לא נוצר — פתח את `/tmp/gpt-image-response.json` ובדוק את שדה ה-`error`.
- שגיאת `invalid_api_key` / `401` → `OPENAI_API_KEY` חסר או לא תקף ב-`.env`.
- שגיאת פרמטרים → בדוק את ה-JSON (size/quality/output_format).
- **אל תשנה את `model`.** הבעיה אינה בשם המודל.
