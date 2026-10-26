# Dietary Policy Validator (Lab 1)

## 1. What is this project

This project is a backend API made with Python FastAPI. It checks a meal plan against a diet policy, like a low-sodium or diabetic hospital policy. The rules are saved in JSON files and not inside the code, so if a hospital wants to change a rule, we only edit the JSON file and the code stays the same.

You send a meal plan and a policy id, then the API tells you if the plan follows the policy, which rules were broken, and a compliance score.

**[SS: /docs page showing GET /policies and POST /validate]**

## 2. Files and folders

The `app` folder has the API code. Inside it, `main.py` has the data models, the policy loading, the rule checking and the endpoints, while `__init__.py` just tells Python that `app` is a package. The `policies` folder has the three JSON policy files: `low_sodium_hospital.json`, `diabetic_hospital.json` and `national_dietary_guidelines.json`. The `examples` folder has sample meal plans (compliant, borderline and non_compliant) that we can use for testing. The `tests` folder has `test_api.py`, which runs the automatic tests. Lastly, `requirements.txt` is the list of libraries to install and `README.md` explains how to run it, with curl and Postman examples.

**[SS: VS Code file explorer showing the folders]**

## 3. Libraries used

- **FastAPI**: makes the web API and the endpoints
- **Uvicorn**: the server that runs the app
- **Pydantic**: checks the data that we send (it comes with FastAPI)
- **pytest**: runs the tests
- **httpx**: lets pytest send fake requests to the API

## 4. What a policy rule looks like

Every rule in a policy file has an id, the nutrient it checks (like `sodium_mg` or `fiber_g`), an operator (`<=`, `>=` or `between`), a threshold, a scope (per meal or per day), a severity (error or warning) and a source where the rule came from. The source is important because the lab is about evidence-based policy, so every rule should point to a real guideline.

**[SS: one policy JSON file open in VS Code]**

## 5. How policies are loaded

When the app needs the policies, it opens the `policies` folder and reads every `.json` file there. Pydantic then checks if the structure is correct. For example, the operator must be one of the three allowed, and a "between" rule must have two numbers in the right order. Each policy is saved using its id, like `diabetic_hospital`. The files are only read one time (cached), not on every request, so it is faster.

## 6. How the operators work

The `<=` operator passes when the actual value is less than or equal to the limit, and `>=` passes when it is more than or equal to the limit. The `between` operator passes when the value is inside the range, and the two ends are included. For example, the diabetic carbs rule is between 45 and 60, so 45 and 60 pass but 44 and 61 fail.

## 7. Per meal vs per day

A per meal rule checks each meal on its own. The carbohydrate rule is like this, so breakfast, lunch and dinner each get their own check. A per day rule adds the nutrient from all foods in all meals first, then checks the total only once. Fiber and added sugar are per day rules.

## 8. How the score is calculated

The score is 100 x (checks passed / total checks). A per meal rule counts once for every meal, and a per day rule counts once. A warning still lowers the score, but only an error makes `compliant` become false.

In my test there were 5 checks in total (3 carb checks, 1 sugar and 1 fiber). Lunch carbs failed and daily fiber failed, so 3 out of 5 passed, which is a score of 60.

## 9. Why missing nutrient data gives an error

If a food doesn't have a nutrient that the policy needs, the app can't know if it is really zero or just not entered. If we count it as zero, an unsafe meal could look okay. That's why the API returns a 422 error and tells which meal, food and nutrient is missing.

## 10. What happens when you send POST /validate

First, FastAPI reads the JSON and Pydantic checks the shape, like if the policy id and meals are there and each meal has foods. Then the app looks for the policy by its id, and if the id doesn't exist it returns 404. Next it checks that every food has the nutrients the policy needs, and if not it returns 422. After that, for every rule it adds up the nutrient (per meal or per day) and compares it with the threshold. Each failed check becomes a violation with the rule id, actual value, allowed value, message, source and severity. At the end it calculates the score and sends back the result.

## 11. Test results

**Test 1: unknown policy id.** I sent the default "string" as the policy id. The API returned 404 and said the policy is unknown, and it listed the three real policies. This shows the error handling works.

**[SS: the 404 response]**

**Test 2: compliant meal plan.** I used the diabetic policy. The carbs were 48, 52 and 55, which are all inside 45 to 60, and the sugar and fiber were also good. The result was `compliant: true`, no violations and a score of 100.

**[SS: the 200 response with score 100]**

**Test 3: non-compliant meal plan.** I changed lunch to 90 carbs, 40 sugar and 1 fiber. The API returned `compliant: false` with two violations. The first one is `meal-carbohydrate-range`, an error, because lunch carbs were 90 and the expected range is 45 to 60. The second one is `daily-fiber-minimum`, a warning, because fiber was 19 for the day and the minimum is 28. The score was 60. There was no sugar violation because the total sugar was 47 g (40 + 5 + 2) and the daily limit is 50 g, so that is correct.

**[SS: the response with violations and score 60]**

**Test 4: automatic tests.** I ran `py -m pytest -q` and all 9 tests passed.

**[SS: terminal showing 9 passed]**

## 12. How to run it

First install the libraries with `py -m pip install -r requirements.txt`. Then start the server with `py -m uvicorn app.main:app --reload` and open `http://127.0.0.1:8000/docs` in the browser. You can test the endpoints there or in Postman.

## 13. Note about the policies

The low-sodium and diabetic hospital policies are example policies, they are not from a real hospital. The national guideline policy is based on the Dietary Guidelines for Americans.
