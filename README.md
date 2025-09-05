from langchain import OpenAI
import pytesseract
from PIL import Image
import pandas as pd

llm = OpenAI(model="gpt-4")

# 1. OCR Receipt
def extract_text_from_receipt(image_path):
    img = Image.open(image_path)
    return pytesseract.image_to_string(img)

# 2. Categorize Expense
def categorize_expense(text):
    prompt = f"Classify this expense into Travel, Meals, or Office: {text}"
    return llm(prompt)

# 3. Validate Expense
def validate_expense(amount, category):
    rules = {"Meals": 1000, "Travel": 5000}
    if amount > rules.get(category, 999999):
        return False, "Exceeds limit"
    return True, "OK"

# 4. Report Generator
def generate_report(expenses, filename="report.xlsx"):
    df = pd.DataFrame(expenses)
    df.to_excel(filename, index=False)

# Orchestrate
expenses = []
receipt_text = extract_text_from_receipt("uber.png")
category = categorize_expense(receipt_text)
is_valid, note = validate_expense(45, category)

expenses.append({"item": receipt_text, "category": category, "amount": 45, "status": note})
generate_report(expenses)
