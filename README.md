# Python写Google爬虫总卡壳？从BeautifulSoup到Selenium再到API方案，哪种才适合你的项目（附踩坑记录和工具选择思路）

如果你在网上搜"python google scraper"，大概率是遇到了下面几种情况之一：写了个`requests + BeautifulSoup`的脚本，跑两次就开始报403；或者用Selenium打开浏览器模拟点击，结果跑得比蜗牛还慢；又或者干脆被验证码搞得脾气都上来了，想知道到底有没有省心点的办法。

这篇就聊聊这件事——不灌水，直接上代码和真实的坑。

## 为什么Google比普通网站难爬

先说清楚一个事实：Google搜索结果页（SERP）不是一个普通的静态页面。它会根据你的IP、地区、设备类型动态调整内容，而且页面结构经常变,今天能用的CSS选择器,过段时间可能就失效了。

更麻烦的是反爬机制。Google会通过这些方式识别和拦截自动化请求：

- **IP频率检测**：同一个IP短时间内发太多请求,直接封
- **验证码（CAPTCHA）**：触发风控后弹出"我不是机器人"验证
- **请求头指纹识别**：没有合理User-Agent、Referer的请求会被秒识别
- **行为模式分析**：没有鼠标移动、滚动、停留时间的"机器式"访问容易被标记

这就是为什么很多人写完第一版爬虫,运行几次就开始全是空白结果或者403错误。

## 方案一：requests + BeautifulSoup（最基础，也最容易翻车）

这是大部分人入门时的第一选择，代码量小，逻辑直观：

python
import requests
from bs4 import BeautifulSoup

headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"
}

params = {"q": "python web scraping", "num": 10}

response = requests.get("https://www.google.com/search", headers=headers, params=params)
soup = BeautifulSoup(response.text, "html.parser")

for result in soup.select("div.g"):
    title = result.select_one("h3")
    if title:
        print(title.get_text())


**问题在哪**：这套代码可能跑个十几次就开始返回验证码页面或者空结果。原因是Google对没有完整浏览器指纹的请求非常敏感，单纯换个User-Agent骗不过去。如果你只是偶尔查几个关键词、量很小，这个方案够用；但凡涉及批量抓取，基本撑不住。

## 方案二：Selenium / Playwright（模拟真实浏览器）

既然Google能识别"不像浏览器"的请求,那就干脆用真浏览器跑：

python
from selenium import webdriver
from selenium.webdriver.common.by import By
import time

driver = webdriver.Chrome()
driver.get("https://www.google.com/search?q=python+web+scraping")
time.sleep(2)

results = driver.find_elements(By.CSS_SELECTOR, "div.g")
for r in results:
    print(r.text)

driver.quit()


这种方式确实能绕过一部分基础检测,因为它是真实渲染JS的浏览器环境。但代价也很明显：

1. **速度慢**：每次请求都要启动/复用浏览器实例，比纯HTTP请求慢好几倍
2. **资源占用高**：批量抓取时开多个浏览器实例,内存和CPU都吃紧
3. **依然会被识别**：Google的Datadome、reCAPTCHA这类防护对Selenium的自动化特征（比如`navigator.webdriver`属性）同样有检测手段
4. **维护成本高**：Google页面结构一变，选择器就得跟着改

适合的场景：抓取量不大、对实时性要求不高、需要处理JS渲染内容的中小型项目。

## 方案三：Google官方Custom Search JSON API

如果你不想碰"爬虫"这个灰色边界，Google自己提供了[Custom Search JSON API](https://developers.google.com/custom-search/v1/overview)，需要在Google Cloud Console申请API Key并配置自定义搜索引擎（CSE）。

python
import requests

API_KEY = "你的API_KEY"
CSE_ID = "你的搜索引擎ID"

url = "https://www.googleapis.com/customsearch/v1"
params = {"key": API_KEY, "cx": CSE_ID, "q": "python web scraping"}

response = requests.get(url, params=params)
data = response.json()

for item in data.get("items", []):
    print(item["title"], item["link"])


优点是合规、稳定，缺点是**免费额度很有限（每天100次查询）**，超出部分按量计费，而且返回的结果字段和真实SERP页面不完全一致，少了一些SEO场景关心的信息（比如完整的相关搜索、PAA问答框等）。

## 方案四：第三方SERP API服务

这是当你真的需要"稳定、大批量、结构化数据"时，多数团队最终会走的路——用专门的SERP抓取服务代替自己维护爬虫逻辑。这类服务的核心思路是：你发一个请求，它们在背后处理IP轮换、验证码绕过、浏览器渲染，直接把结构化的JSON结果还给你。

市面上做这类服务的有不少，比如SerpApi、Oxylabs、Bright Data、Scrape.do，以及**ScraperAPI**。它们的共同逻辑差不多：按请求量或"积分"计费，免费层用于小规模测试，付费层解锁更高并发和地理定位等功能。

拿ScraperAPI来说，它的Google抓取走的是结构化端点，调用方式大致是这样：

python
import requests

API_KEY = "你的API_KEY"
url = "http://api.scraperapi.com"

params = {
    "api_key": API_KEY,
    "url": "https://www.google.com/search?q=python+web+scraping",
    "country_code": "us"
}

response = requests.get(url, params=params)
print(response.text)


它的免费层提供1,000次请求额度（最多5个并发连接），付费从约$49/月的Hobby档起步,往上还有Startup（约$149/月）、Business（约$299/月）等更高并发和更多地区覆盖的档位，定价会随汇率和官网活动有所调整，建议直接看官网当前页面确认。值得注意的是，按它们自己的计费规则，抓Google/Bing这类有反爬保护的页面消耗的积分（约25个/次）比抓普通页面（1个/次）要多不少，预算的时候要把这个系数算进去。

这类服务适合的场景很明确：你需要长期、大批量、稳定地拿到Google SERP数据用于SEO监控、价格追踪、市场调研，又不想自己天天和验证码、IP池打架。如果你想直接体验一下这个方案，可以看看 👉 [ScraperAPI 官网（含优惠信息，本文为联盟推广链接）](https://www.scraperapi.com/?fp_ref=coupons)——这里要坦白说一句，这是一条带联盟参数的链接，如果你通过它注册，我可能会获得一些推广佣金，这不会影响你的实际付费价格。

## 几种方案怎么选——一个简单的对比

| 方案 | 上手难度 | 抗封禁能力 | 成本 | 适合场景 |
|---|---|---|---|---|
| requests + BeautifulSoup | 低 | 弱 | 几乎为零 | 偶尔查几次,数据量很小 |
| Selenium / Playwright | 中 | 中等 | 服务器/本地资源成本 | 需要JS渲染、量不算大 |
| Google官方CSE API | 低 | 无需考虑（官方支持） | 免费额度有限,超量计费 | 合规优先、查询量小 |
| 第三方SERP API（如ScraperAPI等） | 低 | 强（服务商代为处理） | 按请求/积分付费，$49/月起步价位较常见 | 长期、大批量、稳定性优先 |

## 写在最后：别忽视合规这件事

不管选哪种方案，有几点最好提前想清楚：

- **遵守robots.txt和服务条款**：Google的服务条款对自动化查询有明确限制，商业化大规模抓取建议优先考虑官方API或合规的第三方服务,降低账号/IP被封的风险。
- **控制请求频率**：哪怕用了代理池，频率太高也容易触发风控,该加延迟就加延迟。
- **只抓公开数据**：避免涉及个人隐私信息的批量收集。

如果你只是写个脚本练手、跑几十次查询,`requests + BeautifulSoup`完全够用，不用想太多。但一旦涉及到每天定时抓取、监控关键词排名变化、做SEO数据分析这种长期项目，自己维护反爬逻辑的时间成本往往比花钱订阅一个API服务更高——这时候再考虑像ScraperAPI这样的方案，会更划算一些。
