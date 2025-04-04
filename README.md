# 個人理財網站
使用Flask框架搭配MSSQL資料庫，建立個人理財網站，主要功能為管理現金庫存(現金庫存表單)、股票庫存(股票庫存表單)，並畫出股票庫存占比圓餅圖和資產比例占比圓餅圖。  
>![](https://github.com/sujamie/Personal-Finance-Website/blob/main/%E5%80%8B%E4%BA%BA%E7%90%86%E8%B2%A1%E7%B6%B2%E7%AB%99.png)
<hr><br>

現金庫存 : 台幣總額、美金總額、今日匯率、現金總額(換算為台幣)，並有現金更動紀錄  
>![](https://github.com/sujamie/Personal-Finance-Website/blob/main/%E7%8F%BE%E9%87%91%E5%BA%AB%E5%AD%98.png)

股片庫存 : 股票代號、持有股數、目前股價、目前市值、股票資產占比(%)、購買總成本(包含手續費)、平均成本、報酬率(%)
>![](https://github.com/sujamie/Personal-Finance-Website/blob/main/%E8%82%A1%E7%A5%A8%E5%BA%AB%E5%AD%98.png)

股票庫存占比圓餅圖和資產比例占比圓餅圖
>![](https://github.com/sujamie/Personal-Finance-Website/blob/main/%E5%9C%93%E9%A4%85%E5%9C%96.png)

現金庫存表單
>![](https://github.com/sujamie/Personal-Finance-Website/blob/main/%E7%8F%BE%E9%87%91%E5%BA%AB%E5%AD%98%E8%A1%A8%E5%96%AE.png)

股票庫存表單
>![](https://github.com/sujamie/Personal-Finance-Website/blob/main/%E8%82%A1%E7%A5%A8%E5%BA%AB%E5%AD%98%E8%A1%A8%E5%96%AE.png)

## 後端程式碼
``` python
from flask import Flask, render_template, request, g, redirect
import pyodbc as db # 連接 MSSQL
import requests # 發送 HTTP 請求以獲取匯率和股價
import math # 用於數學計算，例如取整數
import matplotlib.pyplot as plt # 用於繪製圖表
import os # 用於檢查和刪除圖檔
import matplotlib
from decimal import Decimal  # 引入 Decimal 模組

# 設定 Matplotlib 以非互動模式運行，避免在伺服器環境中出錯
matplotlib.use('agg')

# 建立 Flask 應用程式
app = Flask(__name__)

# MSSQL 連線設定
DATABASE_CONFIG = {
    "DRIVER": "{SQL Server}",
    "SERVER": "資料庫伺服器名稱", # 資料庫伺服器名稱
    "DATABASE": "資料庫名稱", # 資料庫名稱
    "UID": "使用者名稱", # 使用者名稱
    "PWD": "使用者密碼" # 密碼
}

# 連接 MSSQL
# 取得資料庫連線
# Flask 的 g 物件可用於儲存請求期間的全域變數
def get_db():
    if not hasattr(g, 'mssql_db'):
        conn_str = f"DRIVER={DATABASE_CONFIG['DRIVER']};SERVER={DATABASE_CONFIG['SERVER']};DATABASE={DATABASE_CONFIG['DATABASE']};UID={DATABASE_CONFIG['UID']};PWD={DATABASE_CONFIG['PWD']}"
        # 使用 pyodbc.connect() 來建立一個有效的資料庫連接物件
        g.mssql_db = db.connect(conn_str)
        # 設定編碼
        g.mssql_db.setdecoding(db.SQL_CHAR, encoding='utf-8')
        g.mssql_db.setencoding('utf-8')
    return g.mssql_db

# 釋放資料庫連線
@app.teardown_appcontext
def close_connection(exception):
    if hasattr(g, 'mssql_db'):
        g.mssql_db.close()

# 首頁路由
@app.route('/')
def home():
    conn = get_db()
    cursor = conn.cursor()
    
    # 取得現金資料
    cursor.execute("SELECT * FROM cash")
    cash_result = cursor.fetchall()
    
    # 計算現金總額
    taiwanese_dollars, us_dollars = 0, 0
    for data in cash_result:
        taiwanese_dollars += data[1] # 台幣
        us_dollars += data[2] # 美元

    # 取得匯率資訊
    r = requests.get('https://tw.rter.info/capi.php')
    currency = r.json()

    # 將 us_dollars 和匯率轉換為 Decimal 類型
    us_dollars_converted = Decimal(str(us_dollars)) * Decimal(str(currency['USDTWD']['Exrate']))

    # 計算總資產（台幣 + 美元換算台幣）
    total = math.floor(taiwanese_dollars + us_dollars_converted)

    # 取得股票資料
    cursor.execute("SELECT * FROM stock")
    stock_result = cursor.fetchall()
    
    unique_stock_list = list(set([data[1] for data in stock_result]))
    total_stock_value = 0
    stock_info = []

    # 計算股票資訊
    for stock in unique_stock_list:
        cursor.execute("SELECT * FROM stock WHERE stock_id = ?", (stock,))
        result = cursor.fetchall()

        stock_cost, shares = 0, 0
        for d in result:
            shares += d[2]
            stock_cost += d[2] * d[3] + d[4] + d[5]

        # 取得目前股價
        url = f"https://www.twse.com.tw/exchangeReport/STOCK_DAY?response=json&stockNo={stock}"
        response = requests.get(url)
        data = response.json()
        price_array = data['data']
        current_price = float(price_array[-1][6])

        total_value = round(current_price * shares)
        total_stock_value += total_value
        average_cost = round(stock_cost / shares, 2)
        rate_of_return = round((total_value - stock_cost) * 100 / stock_cost, 2)

        stock_info.append({
            'stock_id': stock,
            'stock_cost': stock_cost,
            'total_value': total_value,
            'average_cost': average_cost,
            'shares': shares,
            'current_price': current_price,
            'rate_of_return': rate_of_return
        })

    for stock in stock_info:
        stock['value_percentage'] = round(stock['total_value'] * 100 / total_stock_value, 2)

    # 股票圓餅圖
    if unique_stock_list:
        labels = tuple(unique_stock_list)
        sizes = [d['total_value'] for d in stock_info]
        fig, ax = plt.subplots(figsize=(6, 5))
        ax.pie(sizes, labels=labels, autopct=None, shadow=None)
        plt.savefig("static/piechart.jpg", dpi=200)
    else:
        if os.path.exists('static/piechart.jpg'):
            os.remove('static/piechart.jpg')

    # 現金 + 股票圓餅圖
    if any([us_dollars, taiwanese_dollars, total_stock_value]):
        labels = ('USD', 'TWD', 'Stock')
        sizes = (float(us_dollars_converted), taiwanese_dollars, total_stock_value)
        fig, ax = plt.subplots(figsize=(6, 5))
        ax.pie(sizes, labels=labels, autopct=None, shadow=None)
        plt.savefig("static/piechart2.jpg", dpi=200)
    else:
        if os.path.exists('static/piechart2.jpg'):
            os.remove('static/piechart2.jpg')

    # 傳遞數據到前端v
    data = {
        'show_pic_1': os.path.exists('static/piechart.jpg'),
        'show_pic_2': os.path.exists('static/piechart2.jpg'),
        'total': total,
        'currency': currency['USDTWD']['Exrate'],
        'ud': us_dollars,
        'td': taiwanese_dollars,
        'cash_result': cash_result,
        'stock_info': stock_info
    }
    return render_template('index.html', data=data)

@app.route('/cash', methods=['GET', 'POST'])
def cash_form():
    taiwanese_dollars = 0
    us_dollars = 0
    if request.method == 'POST':
        # 確保資料轉換為數字
        taiwanese_dollars = request.values.get('taiwanese-dollars', 0)
        us_dollars = request.values.get('us-dollars', 0)
        
        # 嘗試將其轉換為浮點數
        try:
            taiwanese_dollars = float(taiwanese_dollars)  # 確保是數字
        except ValueError:
            taiwanese_dollars = 0  # 如果轉換失敗，預設為 0

        try:
            us_dollars = float(us_dollars)  # 確保是數字
        except ValueError:
            us_dollars = 0  # 如果轉換失敗，預設為 0

        note = request.values['note']
        date = request.values['date']

        # 在插入資料時保證資料型別正確
        conn = get_db()
        cursor = conn.cursor()
        cursor.execute("""
            INSERT INTO cash (taiwanese_dollars, us_dollars, note, date_info) 
            VALUES (?, ?, ?, ?)
        """, (taiwanese_dollars, us_dollars, note, date))
        conn.commit()
        return redirect("/")

    return render_template('cash.html')



@app.route('/cash-delete', methods=['POST'])
def cash_delete():
    transaction_id = request.values['id']
    conn = get_db()
    cursor = conn.cursor()
    cursor.execute("DELETE FROM cash WHERE transaction_id=?", (transaction_id,))
    conn.commit()
    return redirect("/")


@app.route('/stock', methods=['GET', 'POST'])
def stock_form():
    if request.method == 'POST':
        stock_id = request.values.get('stock-id', '').strip()
        stock_num = request.values.get('stock-num', '0').strip()
        stock_price = request.values.get('stock-price', '0').strip()
        processing_fee = request.values.get('processing-fee', '0').strip()
        tax = request.values.get('tax', '0').strip()
        date = request.values.get('date', '').strip()

        # 確保數據類型正確
        try:
            stock_num = int(stock_num) if stock_num else 0
        except ValueError:
            stock_num = 0

        try:
            stock_price = float(stock_price) if stock_price else 0.0
        except ValueError:
            stock_price = 0.0

        try:
            processing_fee = int(processing_fee) if processing_fee else 0
        except ValueError:
            processing_fee = 0

        try:
            tax = int(tax) if tax else 0
        except ValueError:
            tax = 0

        # 連接 MSSQL
        conn = get_db()
        cursor = conn.cursor()
        cursor.execute(""" 
            INSERT INTO stock (stock_id, stock_num, stock_price, processing_fee, tax, date_info) 
            VALUES (?, ?, ?, ?, ?, ?)
        """, (stock_id, stock_num, stock_price, processing_fee, tax, date))
        conn.commit()
        return redirect('/')

    return render_template('stock.html')



if __name__ == '__main__':
    app.run(debug=True)

```
## 前端程式碼
標題動態程式碼
```HTML
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>個人理財頁面</title>
    <link
      href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/css/bootstrap.min.css"
      rel="stylesheet"
      integrity="sha384-T3c6CoIi6uLrA9TneNEoa7RxnatzjcDSCmG1MXxSR1GAsXEV/Dwwykc2MPK8M2HN"
      crossorigin="anonymous"
    />
  </head>
  <body>
    <nav class="navbar navbar-expand-lg bg-body-tertiary">
      <div class="container-fluid">
        <button
          class="navbar-toggler"
          type="button"
          data-bs-toggle="collapse"
          data-bs-target="#navbarNav"
          aria-controls="navbarNav"
          aria-expanded="false"
          aria-label="Toggle navigation"
        >
          <span class="navbar-toggler-icon"></span>
        </button>
        <div class="collapse navbar-collapse" id="navbarNav">
          <a class="navbar-brand" href="#">個人理財網站</a>
          <ul class="navbar-nav">
            <li class="nav-item">
              <a class="nav-link" href="/">首頁</a>
            </li>
            <li class="nav-item">
              <a class="nav-link" href="/cash">現金庫存表單</a>
            </li>
            <li class="nav-item">
              <a class="nav-link" href="/stock">股票庫存表單</a>
            </li>
          </ul>
        </div>
      </div>
    </nav>

    <div id="content" style="padding: 2rem">
      {% block content %}{% endblock %}
    </div>

    <script
      src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/js/bootstrap.bundle.min.js"
      integrity="sha384-C6RzsynM9kWDrMNeT87bh95OGNyZPhcTNXj1NW7RuBCsyN/o0jlpcV8Qyq46cDfL"
      crossorigin="anonymous"
    ></script>
  </body>
</html>
```
首頁程式碼
```HTML
{% extends "base.html" %} {% block content %}
<div id="cash-info">
  <h2>現金庫存</h2>
  <table class="table table-bordered">
    <tr>
      <td>台幣總額: {{data['td']}} 元</td>
      <td>美金總額: {{data['ud']}} 元</td>
      <td>
        今日匯率 (出處:全球即時匯率API
        ): {{data['currency']}}
      </td>
      <td>現金總額: {{data['total']}} 元</td>
    </tr>
  </table>

  <h4>現金更動紀錄</h4>
  <table class="table table-bordered">
    <thead>
      <tr>
        <th scope="col">台幣</th>
        <th scope="col">美金</th>
        <th scope="col">註記</th>
        <th scope="col">時間</th>
        <th scope="col">刪除資料</th>
      </tr>
    </thead>
    <tbody>
      {% for data in data['cash_result'] %}
      <tr>
        <td>{{data[1]}}</td>
        <td>{{data[2]}}</td>
        <td>{{data[3]}}</td>
        <td>{{data[4]}}</td>
        <td>
          <form action="/cash-delete" method="post">
            <input type="hidden" name="id" value="{{data[0]}}" />
            <button class="btn btn-primary">刪除此筆資料</button>
          </form>
        </td>
      </tr>
      {% endfor %}
    </tbody>
  </table>
</div>

<hr />

<div id="stock-info">
  <h2>股票庫存</h2>
  <table class="table table-bordered">
    <thead>
      <tr>
        <th scope="col">股票代號</th>
        <th scope="col">持有股數</th>
        <th scope="col">目前股價</th>
        <th scope="col">目前市值</th>
        <th scope="col">股票資產占比(%)</th>
        <th scope="col">購買總成本(包含手續費)</th>
        <th scope="col">平均成本</th>
        <th scope="col">報酬率(%)</th>
      </tr>
    </thead>
    <tbody>
      {% for d in data['stock_info'] %}
      <tr>
        <td>{{d['stock_id']}}</td>
        <td>{{d['shares']}}</td>
        <td>{{d['current_price']}}</td>
        <td>{{d['total_value']}}</td>
        <td>{{d['value_percentage']}}</td>
        <td>{{d['stock_cost']}}</td>
        <td>{{d['average_cost']}}</td>
        <td>{{d['rate_of_return']}}</td>
      </tr>
      {% endfor %}
    </tbody>
  </table>
</div>

<div id="chart" style="display: flex; flex-wrap: wrap">
  {% if data['show_pic_1'] %}
  <figure style="flex: 0 1 500px; margin: 10px">
    <figcaption>股票庫存占比圖</figcaption>
    <img style="width: 100%" src="/static/piechart.jpg" alt="股票庫存占比圖" />
  </figure>
  {% endif %} {% if data['show_pic_2'] %}
  <figure style="flex: 0 1 500px; margin: 10px">
    <figcaption>資產比例占比圖</figcaption>
    <img style="width: 100%" src="/static/piechart2.jpg" alt="資產比例占比圖" />
  </figure>
  {% endif %}
</div>
{% endblock %}
```
現金庫存表單程式碼
```HTML
{% extends "base.html" %} {% block content %}
<form method="post" action="http://127.0.0.1:5000/cash">
  <div class="mb-3">
    <label for="taiwanese-dollars" class="form-label">台幣</label>
    <input
      type="number"
      class="form-control"
      id="taiwanese-dollars"
      name="taiwanese-dollars"
    />
  </div>

  <div class="mb-3">
    <label for="us-dollars" class="form-label">美金</label>
    <input
      step="0.01"
      type="number"
      class="form-control"
      id="us-dollars"
      name="us-dollars"
    />
  </div>

  <div class="mb-3">
    <label for="note" class="form-label">備註</label>
    <input type="text" class="form-control" id="note" name="note" />
  </div>

  <div class="mb-3">
    <label for="date" class="form-label">日期</label>
    <input type="date" class="form-control" id="date" name="date" />
  </div>

  <button type="submit" class="btn btn-primary">提交表單</button>
</form>
{% endblock %}
```
股票庫存表單程式碼
```HTML
{% extends "base.html" %} {% block content %}
<form method="post" action="http://127.0.0.1:5000/stock">
  <div class="mb-3">
    <label for="stock-id" class="form-label">股票代號</label>
    <input type="text" class="form-control" id="stock-id" name="stock-id" />
  </div>

  <div class="mb-3">
    <label for="stock-num" class="form-label">成交股數</label>
    <input type="number" class="form-control" id="stock-num" name="stock-num" />
  </div>

  <div class="mb-3">
    <label for="stock-price" class="form-label">成交單價</label>
    <input
      type="number"
      step="0.01"
      class="form-control"
      id="stock-price"
      name="stock-price"
    />
  </div>

  <div class="mb-3">
    <label for="processing-fee" class="form-label">手續費</label>
    <input
      type="number"
      class="form-control"
      id="processing-fee"
      name="processing-fee"
    />
  </div>

  <div class="mb-3">
    <label for="tax" class="form-label">交易稅</label>
    <input type="number" class="form-control" id="tax" name="tax" />
  </div>

  <div class="mb-3">
    <label for="date" class="form-label">日期</label>
    <input type="date" class="form-control" id="date" name="date" />
  </div>

  <button type="submit" class="btn btn-primary">提交表單</button>
</form>
{% endblock %}
```
