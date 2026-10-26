# Dietary-policy-validator

A beginner-friendly FastAPI service that checks meal plans against dietary policies stored as JSON files. It sums nutrient values from foods, checks per-meal and per-day rules, and reports errors separately from warnings.

## Run locally

```powershell
py -m venv .venv
.\.venv\Scripts\Activate.ps1
py -m pip install -r requirements.txt
uvicorn app.main:app --reload
```

The API documentation is available at <http://127.0.0.1:8000/docs>.

## Policies and rule behavior

Policy JSON files are in `policies/`. Each rule specifies an ID, nutrient name, operator, threshold, scope, severity, and source citation. The included policies are:

- `low_sodium_hospital`: no more than 2000 mg sodium per day.
- `diabetic_hospital`: 45–60 g carbohydrates per meal, no more than 50 g added sugar per day, and at least 28 g fiber per day.
- `national_dietary_guidelines`: no more than 2300 mg sodium per day, based on the U.S. Dietary Guidelines for Americans.

The diabetic policy's sugar and fiber thresholds use a 2000-kcal reference pattern; individual clinical targets can differ. The carbohydrate range is a hospital target, not a universal prescription. Values exactly on a threshold pass. Missing nutrients required by a policy are rejected with HTTP 422 rather than assumed to be zero. A warning is reported and reduces the score, but only an error makes the result non-compliant.

The compliance score is the percentage of individual rule checks that pass. A per-meal rule is checked once for each meal; a per-day rule is checked once for the full plan.

## Validate a meal plan

```powershell
curl.exe -X POST http://127.0.0.1:8000/validate `
  -H "Content-Type: application/json" `
  --data-binary "@examples/compliant.json"
```

Example response:

```json
{
  "compliant": true,
  "violations": [],
  "compliance_score": 100.0
}
```

Each food reports its nutrient amounts; the service adds those values to evaluate each meal and the full day. For example, a borderline plan is sent the same way:

```powershell
curl.exe -X POST http://127.0.0.1:8000/validate `
  -H "Content-Type: application/json" `
  --data-binary "@examples/borderline.json"
```

The request body can also be created directly in Postman using **POST** `http://127.0.0.1:8000/validate`, selecting **Body → raw → JSON**, and entering:

```json
{
  "policy_id": "low_sodium_hospital",
  "meals": [
    {
      "name": "Lunch",
      "foods": [
        {
          "name": "Soup",
          "nutrients": { "sodium_mg": 850 }
        },
        {
          "name": "Sandwich",
          "nutrients": { "sodium_mg": 600 }
        }
      ]
    }
  ]
}
```

List available policies with **GET** `http://127.0.0.1:8000/policies`.

## Sample plans and tests

`examples/compliant.json`, `examples/borderline.json`, and `examples/non_compliant.json` are ready-to-send diabetic hospital policy requests.

```powershell
py -m pytest
```
