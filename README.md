import time
import alpaca_trade_api as tradeapi
from newsapfrom langchain import OpenAI
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
generate_report(expenses)i import NewsApiClient
from openai import OpenAI

# ---- Config ----
ALPACA_API_KEY = "YOUR_ALPACA_API_KEY"
ALPACA_SECRET_KEY = "YOUR_ALPACA_SECRET_KEY"
BASE_URL = "https://paper-api.alpaca.markets"

NEWS_API_KEY = "YOUR_NEWSAPI_KEY"

client = OpenAI(api_key="YOUR_OPENAI_API_KEY")
api = tradeapi.REST(ALPACA_API_KEY, ALPACA_SECRET_KEY, BASE_URL, api_version="v2")
newsapi = NewsApiClient(api_key=NEWS_API_KEY)


# ---- TOOL 1: Fetch stock price ----
def get_stock_price(symbol="AAPL"):
    barset = api.get_bars(symbol, "1Min", limit=1)
    return barset[0].c


# ---- TOOL 2: Fetch latest stock news ----
def get_stock_news(symbol="Apple"):
    articles = newsapi.get_everything(q=symbol, language="en", sort_by="publishedAt", page_size=5)
    if not articles["articles"]:
        return "No recent news."
    summaries = []
    for a in articles["articles"]:
        summaries.append(f"- {a['title']}: {a['description']}")
    return "\n".join(summaries)


# ---- TOOL 3: Place trade ----
def place_trade(symbol, action, qty):
    if action.upper() == "BUY":
        api.submit_order(symbol=symbol, qty=qty, side="buy", type="market", time_in_force="gtc")
    elif action.upper() == "SELL":
        api.submit_order(symbol=symbol, qty=qty, side="sell", type="market", time_in_force="gtc")
    print(f"💹 Executed {action} for {qty} shares of {symbol}")


# ---- LLM Reasoning with RAG ----
def decide_action(price, news_summary, symbol="AAPL", threshold=145):
    prompt = f"""
    You are an AI trading agent.

    DATA:
    - Stock {symbol} current price: {price} USD
    - Buy threshold: {threshold}
    - News summary:
    {news_summary}

    INSTRUCTIONS:
    Analyze BOTH price and news.
    If news is positive + price < {threshold} → BUY.
    If news is negative + price > {threshold + 5} → SELL.
    If mixed or uncertain, just notify.
    
    Respond with exactly one of: "buy", "sell", "notify", "none".
    """
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}]
    )
    return response.choices[0].message.content.strip().lower()


# ---- Agent Loop ----
def run_agent():
    while True:
        try:
            price = get_stock_price("AAPL")
            news = get_stock_news("Apple")

            print(f"\n📈 Price: {price}")
            print(f"📰 News:\n{news[:300]}...")  # show snippet

            decision = decide_action(price, news)

            if "buy" in decision:
                place_trade("AAPL", "BUY", 10)
                print("📢 Agent decided to BUY 10 shares of AAPL.")
            elif "sell" in decision:
                place_trade("AAPL", "SELL", 10)
                print("📢 Agent decided to SELL 10 shares of AAPL.")
            elif "notify" in decision:
                print(f"📢 Price/News alert: AAPL at {price}")
            else:
                print("🤖 Agent decided to do nothing this cycle.")
        except Exception as e:
            print("⚠️ Error:", e)

        time.sleep(120)  # check every 2 minutes


if __name__ == "__main__":
    run_agent()
