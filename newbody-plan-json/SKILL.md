---
name: newbody-plan-json
description: Output NewBody APP importable JSON for combined diet+exercise weekly plans with a shopping list. Use when the user asks to generate, format, convert, validate, or provide a JSON plan for NewBody, NewBody APP import, combined diet and exercise plans, weekly meal plans, workout plans, or采购/食材清单.
---

# NewBody Plan JSON

## Output Rule

When generating a plan for NewBody APP import, output **valid JSON only** unless the user explicitly asks for explanation. The APP's import page auto-strips ` ```json ``` ` fences and extracts the first `{…}` block, so raw JSON or JSON inside a code block both work. Do **not** add comments, ellipses, trailing commas, summaries, or extra prose.

## Top-Level Structure

Only support combined diet + exercise plans. Always include a shopping list for procurement.

```json
{
  "diet_plan":     { "days": [ /* DayPlan × 7 */ ] },
  "exercise_plan": { "days": [ /* ExerciseDayPlan × 7 */ ] },
  "shopping_list": [ /* ShoppingListItem */ ]
}
```

Use these exact top-level keys:
- `diet_plan`
- `exercise_plan`
- `shopping_list`

Do not output diet-only or exercise-only JSON. Do not wrap in an outer `"plan"` key.

---

## Diet Plan: DayPlan

```json
{
  "day": "周一",
  "meals": {
    "breakfast": [ MealItem ],
    "lunch":     [ MealItem ],
    "dinner":    [ MealItem ],
    "snack":     [ MealItem ]
  },
  "exercise":     [],
  "total_intake": 1210,
  "total_burn":   0,
  "note":         ""
}
```

### MealItem

```json
{
  "food": "全麦吐司",
  "amount": "2片",
  "cal": 160,
  "method": "直接食用或轻微烤热"
}
```

- `food`: string
- `amount`: string，自由描述（"150g" / "2片" / "一碗" 都行）
- `cal`: **int**，kcal
- `method`: string，餐食简易做法，用一句话说明怎么处理；无需烹饪的食物写 `"直接食用"`。

### Rules

- `meals` keys **固定** `breakfast` / `lunch` / `dinner` / `snack`，写中文 key 前端读不到。
- 无食物的餐用空数组 `[]`。
- 每个 `MealItem` 必须包含 `food` / `amount` / `cal` / `method`。
- `exercise` 在饮食计划里通常留空 `[]`。
- `total_intake` / `total_burn`: **int**，不能是字符串。
- `note`: 可选，空字符串即可。

### Diet Generation Constraints

Unless the user explicitly overrides these preferences:

- 饮食计划里不要出现鱼类或鱼制品，包括但不限于鱼、巴沙鱼、龙利鱼、带鱼、鲈鱼、鳕鱼、金枪鱼、三文鱼、鱼丸、鱼豆腐、鱼汤。
- 午餐不要出现汤、汤面、汤粉、粥、羹、炖汤；普通带汤汁的菜可以出现，但不要把汤本身作为午餐项目。
- 午餐优先使用米饭/杂粮饭 + 鸡胸肉/鸡腿肉/瘦牛肉/虾仁/鸡蛋/豆腐等非鱼蛋白 + 西兰花/芦笋/西葫芦/菠菜/黄瓜/生菜等便携蔬菜。
- 如果用户提供的原计划包含鱼或午餐汤，改写时应替换为非鱼蛋白和无汤午餐，而不是原样保留。

---

## Exercise Plan: ExerciseDayPlan

```json
{
  "day":   "周一",
  "type":  "训练日",
  "focus": "有氧",
  "exercises": [ ExercisePlanEntry ],
  "total_cal": 250,
  "note":      "有氧日"
}
```

### ExercisePlanEntry

```json
{
  "name":      "椭圆机匀速",
  "equipment": "椭圆机",
  "sets":      1,
  "reps":      "30分钟",
  "rest":      "-",
  "cal":       250,
  "tip":       "心率维持130"
}
```

- `name`: string，动作名
- `equipment`: string，无器械写 `"徒手"`
- `sets`: **int**
- `reps`: string，自由描述（"12次" / "30分钟" / "每侧10次" 都行）
- `rest`: string，组间休息；无意义时写 `"-"`
- `cal`: **int**，单个动作预计消耗
- `tip`: 可选，技术提示

### Rest day

```json
{
  "day": "周日",
  "type": "休息日",
  "focus": "恢复",
  "exercises": [],
  "total_cal": 0,
  "note": "休息"
}
```

---

## Shopping List: ShoppingListItem

`shopping_list` 用于采购，必须根据 7 天饮食计划汇总生成。不要列运动器材、调味品以外的非食材物品，除非用户明确要求。

```json
{
  "category": "蛋白质",
  "item":     "鸡胸肉",
  "amount":   "500g",
  "note":     "周一午餐、周四午餐"
}
```

- `category`: string，建议使用 `蛋白质` / `主食` / `蔬菜` / `水果` / `乳制品` / `调味品` / `其他`。
- `item`: string，食材名称。
- `amount`: string，采购量，可用 `"500g"` / `"14个"` / `"2盒"` / `"适量"`。
- `note`: string，可写用途、分装方式、替代选择；没有就写空字符串。

### Shopping List Rules

- `shopping_list` 必须覆盖 7 天饮食计划中出现的主要食材。
- 同类同名食材应合并成一项，采购量按一周总量估算。
- 不要把成品餐名当成采购项，例如不要写 `"鸡胸肉饭盒"`，应拆成 `"鸡胸肉"`、`"米饭"`、`"西兰花"`。
- 默认不要列鱼类或鱼制品；如果用户明确要求鱼，才允许出现在采购清单。

---

## Hard Constraints (违反则导入失败)

1. **顶层必须是 JSON 对象**，不能是数组。
2. 顶层必须且只使用 `diet_plan` / `exercise_plan` / `shopping_list` 三个 key。
3. **`diet_plan.days` 和 `exercise_plan.days` 必须是数组**，各 7 项。
4. **`day` 值必须是** `周一` / `周二` / `周三` / `周四` / `周五` / `周六` / `周日`——英文或数字无法匹配。
5. **`meals` keys 只识别** `breakfast` / `lunch` / `dinner` / `snack`。
6. 每个 `MealItem` 必须包含 `food` / `amount` / `cal` / `method`。
7. **整数字段**（`cal` / `total_intake` / `total_burn` / `total_cal` / `sets`）必须是数字，不能是字符串。
8. `shopping_list` 必须是数组，且每项必须包含 `category` / `item` / `amount` / `note`。
9. 不支持只输出饮食计划或只输出运动计划。

## Validity Checklist

Before outputting, mentally verify:

- Exactly 7 days per included plan, `周一` through `周日`.
- Top-level object contains exactly `diet_plan`, `exercise_plan`, and `shopping_list`.
- `diet_plan.days` has exactly 7 days and `exercise_plan.days` has exactly 7 days.
- All integer fields are numbers, not strings.
- No extra keys (`plan_name`, `level`, `summary`, `metadata`).
- No comments, no placeholder ellipses, no trailing commas.
- `meals` uses only the four English keys.
- Every meal item includes a concise `method` field.
- `total_intake` equals the sum of all `cal` in that day's meals.
- `total_cal` equals the sum of all `cal` in that day's exercises.
- `shopping_list` covers the main ingredients used across all 7 days.
- Same ingredient is merged into one shopping item with a weekly total amount.
- Diet output contains no fish or fish products unless explicitly requested by the user.
- Lunch output contains no soup, porridge, broth, soup noodles, or soup as a standalone item unless explicitly requested by the user; saucy dishes are allowed.
