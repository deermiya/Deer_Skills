---
name: newbody-plan-json
description: Output NewBody APP importable JSON for diet plans, exercise plans, or combined diet+exercise weekly plans. Use when the user asks to generate, format, convert, validate, or provide a JSON plan for NewBody, NewBody APP import, diet_plan, exercise_plan, weekly meal plans, workout plans, or when they mention handing the format to another AI.
---

# NewBody Plan JSON

## Output Rule

When generating a plan for NewBody APP import, output **valid JSON only** unless the user explicitly asks for explanation. The APP's import page auto-strips ` ```json ``` ` fences and extracts the first `{…}` block, so raw JSON or JSON inside a code block both work. Do **not** add comments, ellipses, trailing commas, or unnecessary summaries. (Note: Meal prep workflows should now be included INSIDE the JSON structure under `prep_workflow`).

## Top-Level Structure (三选一)

### A. Combined plan (recommended)

```json
{
  "diet_plan":     { "days": [ /* DayPlan × 7 */ ] },
  "exercise_plan": { "days": [ /* ExerciseDayPlan × 7 */ ] }
}
```

Key name aliases accepted by the APP (pick one, don't mix):
- Diet: `diet_plan` / `dietPlan` / `meal_plan` / `mealPlan` / `food_plan` / `foodPlan`
- Exercise: `exercise_plan` / `exercisePlan` / `workout_plan` / `workoutPlan`

Wrapping in an outer `"plan"` key is also accepted.

### B. Diet-only

```json
{ "days": [ /* DayPlan × 7 */ ] }
```

APP auto-detects when `days[0]` contains `meals` / `total_intake` / `total_burn`.

### C. Exercise-only

```json
{ "days": [ /* ExerciseDayPlan × 7 */ ] }
```

APP auto-detects when `days[0]` contains `exercises` / `total_cal`.

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
  "food": "鸡胸肉炒西兰花",
  "amount": "150g+200g",
  "cal": 320,
  "method": "1.鸡丁加黑胡椒腌10分 2.西兰花焯水2分 3.少油快炒加生抽调味"
}
```

- `food`: string
- `amount`: string，自由描述（"150g" / "2片" / "一碗" 都行）
- `cal`: **int**，kcal
- `method`: string，简易做法。要求：1) 细节丰富且**按条列出**(如: 1.xx 2.xx)；2) 步骤中**必须包含预估分钟时长和具体调料**(如: 焯水2分、加黑胡椒生抽)；3) 总体**严格控制在50字以内**(避免弹窗过长)；4) 无需烹饪的食物写 `"直接食用"`。

### Rules

- `meals` keys **固定** `breakfast` / `lunch` / `dinner` / `snack`，写中文 key 前端读不到。
- 无食物的餐用空数组 `[]`。
- `exercise` 在饮食计划里通常留空 `[]`。
- `total_intake` / `total_burn`: **int**，不能是字符串。
- `note`: 可选，空字符串即可。

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

## Hard Constraints (违反则导入失败)

1. **顶层必须是 JSON 对象**，不能是数组。
2. **`days` 必须是数组**且至少一项。
3. **`day` 值必须是** `周一` / `周二` / `周三` / `周四` / `周五` / `周六` / `周日`——英文或数字无法匹配。
4. **`meals` keys 只识别** `breakfast` / `lunch` / `dinner` / `snack`。
5. **整数字段**（`cal` / `total_intake` / `total_burn` / `total_cal` / `sets`）必须是数字，不能是字符串。
6. 生成完整 7 天；缺天不报错，但当天看不到计划。

## Validity Checklist

Before outputting, mentally verify:

- Exactly 7 days per included plan, `周一` through `周日`.
- All integer fields are numbers, not strings.
- No extra keys (`plan_name`, `level`, `summary`, `metadata`).
- No comments, no placeholder ellipses, no trailing commas.
- `meals` uses only the four English keys.
- `total_intake` equals the sum of all `cal` in that day's meals.
- `total_cal` equals the sum of all `cal` in that day's exercises.
- Every meal item includes a concise `method` field.

## Prep Workflow (Optional JSON field)

If the user requests a meal prep plan or if generating a weekly strategy, include a `prep_workflow` array at the top level of the JSON. Each item must have `step` and `detail`.

```json
{
  "diet_plan": { "days": [...] },
  "prep_workflow": [
    { "step": "1. 煮蛋 (0分钟开始)", "detail": "14个蛋冷水下锅，大火煮开转小火8分钟，捞出冰水冷却" },
    { "step": "2. 蒸米饭 (0分钟开始)", "detail": "500g生米洗净，电饭煲启动" },
    { "step": "3. 焯蔬菜 (同时进行)", "detail": "烧一大锅水，依次焯西兰花(2分钟)、芦笋(1分钟)、菠菜(30秒)，捞出沥干" },
    { "step": "4. 处理蛋白质 (20分钟)", "detail": "切肉、切鱼、分装、腌制、贴标签" },
    { "step": "5. 分装 (50分钟开始)", "detail": "蔬菜分份装盒，米饭分份装盒，全部贴好日期标签" },
    { "step": "6. 入库 (70分钟)", "detail": "冷藏区放前3天的，冷冻区放后面的" }
  ]
}
```
