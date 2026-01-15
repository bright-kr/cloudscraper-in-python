# Using cloudscraper in Python

[![Bright Data Promo](https://github.com/bright-kr/LinkedIn-Scraper/raw/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner.png)](https://brightdata.co.kr/)

이 가이드는 `cloudscraper` Python 라이브러리를 사용하여 Cloudflare의 보호를 우회하고 오류를 처리하는 방법을 설명합니다:

- [Install Prerequisites](#install-prerequisites)
- [Write Initial Scraping Code](#write-initial-scraping-code)
- [Incorporate cloudscraper](#incorporate-cloudscraper)
- [Use Additional cloudscraper Features](#use-additional-cloudscraper-features)
  - [Proxies](#proxies)
  - [Change the User Agent and JavaScript Interpreter](#change-the-user-agent-and-javascript-interpreter)
  - [Handling CAPTCHAs](#handling-captchas)
- [Common cloudscraper Errors](#common-cloudscraper-errors)
  - [`module not found`](#module-not-found)
  - [`cloudscraper can’t bypass the latest Cloudflare version`](#cloudscraper-cant-bypass-the-latest-cloudflare-version)
- [cloudscraper Alternatives](#cloudscraper-alternatives)
- [Conclusion](#conclusion)

## Install Prerequisites

Python 3가 설치되어 있는지 확인한 다음, 필요한 패키지를 설치합니다:

```bash
pip install tqdm==4.66.5 requests==2.32.3 beautifulsoup4==4.12.3
```

## Write Initial Scraping Code

이 가이드는 [ChannelsTV website](https://www.channelstv.com/)에서 특정 날짜에 게시된 뉴스 기사로부터 메타데이터를 스크레이핑한다고 가정합니다. 아래는 초기 Python 스크립트입니다:

```python
import requests
from bs4 import BeautifulSoup
from datetime import datetime
from tqdm.auto import tqdm

def extract_article_data(article_source, headers):
    response = requests.get(article_source, headers=headers)
    if response.status_code != 200:
        return None

    soup = BeautifulSoup(response.content, 'html.parser')

    title = soup.find(class_="post-title display-3").text.strip()

    date = soup.find(class_="post-meta_time").text.strip()
    date_object = datetime.strptime(date, 'Updated %B %d, %Y').date()

    categories = [category.text.strip() for category in soup.find('nav', {"aria-label": "breadcrumb"}).find_all('li')]

    tags = [tag.text.strip() for tag in soup.find("div", class_="tags").find_all("a")]

    article_data = {
        'date': date_object,
        'title': title,
        'link': article_source,
        'tags': tags,
        'categories': categories
    }

    return article_data

def process_page(articles, headers):
    page_data = []
    for article in tqdm(articles):
        url = article.find('a', href=True).get('href')
        if "https://" not in url:
            continue
        article_data = extract_article_data(url, headers)
        if article_data:
            page_data.append(article_data)
    return page_data

def scrape_articles_per_day(base_url, headers):
    day_data = []
    page = 1

    while True:
        page_url = f"{base_url}/page/{page}"
        response = requests.get(page_url, headers=headers)

        if not response or response.status_code != 200:
            break

        soup = BeautifulSoup(response.content, 'html.parser')
        articles = soup.find_all('article')

        if not articles:
            break
        page_data = process_page(articles, headers)
        day_data.extend(page_data)

        page += 1

    return day_data

headers = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36',
}

URL = "https://www.channelstv.com/2024/08/01/"

scraped_articles = scrape_articles_per_day(URL, headers)
print(f"{len(scraped_articles)} articles were scraped.")
print("Samples:")
print(scraped_articles[:2])
```

이 스크립트는 스크레이핑을 위한 세 가지 핵심 함수를 정의합니다. `extract_article_data` 함수는 기사 웹페이지에서 콘텐츠를 가져와 제목, 게시 날짜, 태그, 카테고리와 같은 메타데이터를 딕셔너리로 추출합니다.

다음으로 `process_page` 함수는 특정 페이지에 있는 모든 기사를 순회하면서 `extract_article_data`를 사용해 메타데이터를 추출하고, 결과를 리스트로 취합합니다.

마지막으로 `scrape_articles_per_day` 함수는 페이지네이션된 결과를 체계적으로 탐색하며, 더 이상 페이지가 없을 때까지 `while` 루프에서 페이지 번호를 증가시킵니다.

스크레이퍼를 실행하기 위해 스크립트는 2024년 8월 1일이라는 필터링 날짜가 포함된 대상 URL을 지정합니다. User-Agent 헤더를 설정한 뒤, 제공된 URL과 헤더로 `scrape_articles_per_day` 함수를 호출합니다. 스크레이핑된 전체 기사 수가 출력되며, 처음 두 개 결과의 미리보기도 함께 출력됩니다.

그러나 ChannelsTV website는 Cloudflare 보호를 사용하므로, `extract_article_data` 및 `scrape_articles_per_day`에서 수행되는 직접적인 HTTP 리クエ스트가 차단되어 스크립트가 예상대로 동작하지 않습니다.

스크립트를 실행하면 일반적으로 출력은 다음과 같이 나타납니다:

```
0 articles were scraped.
Samples:
[]
```

## Incorporate cloudscraper

Cloudflare를 우회하기 위해 `cloudscraper`를 설치합니다:

```bash
pip install cloudscraper==1.2.71
```

`cloudscraper`를 사용하도록 스크립트를 수정합니다:

```python
import cloudscraper

def fetch_html_content(url, headers):
    try:
        scraper = cloudscraper.create_scraper()
        response = scraper.get(url, headers=headers)

        if response.status_code == 200:
            return response
        else:
            print(f"Failed to fetch URL: {url}. Status code: {response.status_code}")
            return None
    except Exception as e:
        print(f"An error occurred while fetching URL: {url}. Error: {str(e)}")
        return None
```

이 함수 `fetch_html_content`는 URL과 리クエ스트 헤더를 입력으로 받습니다. `cloudscraper.create_scraper()`를 사용해 웹페이지를 가져오려고 시도합니다. 리クエ스트가 성공(상태 코드 200)하면 응답를 반환하고, 그렇지 않으면 오류 메시지를 출력한 뒤 `None`을 반환합니다. 예외가 발생하면 오류를 캐치하여 표시한 다음 `None`을 반환합니다.

이 업데이트 이후에는 모든 `requests.get` 호출을 `fetch_html_content`로 교체하여 Cloudflare로 보호되는 website와의 호환성을 보장합니다. 첫 번째 수정은 위에示된 것처럼 `extract_article_data` 함수에서 이루어집니다.

이제 스크레이핑 함수에서 `requests.get` 호출을 `fetch_html_content`로 교체합니다.

```python
def extract_article_data(article_source, headers):
    response = fetch_html_content(article_source, headers)
```

그 다음, `scrape_articles_per_day` 함수에서 `requests.get` 호출을 다음과 같이 교체합니다:

```python
def scrape_articles_per_day(base_url, headers):
    day_data = []
    page = 1

    while True:
        page_url = f"{base_url}/page/{page}" 
        response = fetch_html_content(page_url, headers)
```

이 함수를 정의하면 cloudscraper 라이브러리가 Cloudflare의 제한을 회피하는 데 도움이 됩니다.

코드를 실행하면 출력은 다음과 같이 표시됩니다:

```
Failed to fetch URL: https://www.channelstv.com/2024/08/01//page/5. Status code: 404
55 articles were scraped.
Samples:
[{'date': datetime.date(2024, 8, 1),
  'title': 'Resilience, Tear Gas, Looting, Curfew As #EndBadGovernance Protests Hold',
  'link': 'https://www.channelstv.com/2024/08/01/tear-gas-resilience-looting-curfew-as-endbadgovernance-protests-hold/',
  'tags': ['Eagle Square', 'Hunger', 'Looting', 'MKO Abiola Park', 'violence'],
  'categories': ['Headlines']},
 {'date': datetime.date(2024, 8, 1),
  'title': 'Mother Of Russian Artist Freed In Prisoner Swap Waiting To 'Hug' Her',
  'link': 'https://www.channelstv.com/2024/08/01/mother-of-russian-artist-freed-in-prisoner-swap-waiting-to-hug-her/',
  'tags': ['Prisoner Swap', 'Russia'],
  'categories': ['World News']}]
```

## Use Additional cloudscraper Features

### Proxies

cloudscraper를 사용하면 프록시를 정의하고, 이미 생성한 `cloudscraper` 객체에 다음과 같이 전달할 수 있습니다:

```python
scraper = cloudscraper.create_scraper()
proxy = {
    'http': 'http://your-proxy-ip:port',
    'https': 'https://your-proxy-ip:port'
}
response = scraper.get(URL, proxies=proxy)
```

여기서는 기본 값으로 스크레이퍼 객체를 먼저 정의합니다. 그런 다음 `http` 및 `https` 프록시가 포함된 프록시 딕셔너리를 정의합니다. 이후 일반적인 `request.get` 메서드에서처럼 `scraper.get` 메서드에 `proxies`로 프록시 딕셔너리 객체를 전달합니다.

### Change the User Agent and JavaScript Interpreter

cloudscraper 라이브러리는 user agent를 자동 생성할 수 있으며, 스크레이퍼에서 사용할 JavaScript interpreter와 엔진을 지정할 수 있게 해줍니다. 다음은 예시 코드입니다:

```python
scraper = cloudscraper.create_scraper(
    interpreter="nodejs",
    browser={
        "browser": "chrome",
        "platform": "ios",
        "desktop": False,
    }
)
```

위 스크립트는 interpreter를 `"nodejs"`로 설정하고, browser 매개변수에 딕셔너리를 전달합니다. 브라우저는 Chrome으로, 플랫폼은 `"ios"`로 설정됩니다. desktop 매개변수는 `False`로 설정되어 브라우저가 모바일에서 실행됨을 시사합니다.

### Handling CAPTCHAs

cloudscraper 라이브러리는 reCAPTCHA, hCaptcha 등을 우회하기 위해 서드파티 CAPTCHA solver를 지원합니다. 다음 스니펫은 CAPTCHA를 처리하도록 스크레이퍼를 수정하는 방법을 보여줍니다:

```python
scraper = cloudscraper.create_scraper(
  captcha={
    'provider': 'capsolver',
    'api_key': 'your_capsolver_api_key'
  }
)
```

이 코드는 CAPTCHA provider로 Capsolver를 사용하며 Capsolver API key를 사용합니다. 두 값은 딕셔너리에 저장되어 `cloudscraper.create_scraper` 메서드의 CAPTCHA 매개변수로 전달됩니다.

## Common cloudscraper Errors

### `module not found`

`cloudscraper`가 설치되어 있는지 확인합니다:

```sh
pip install cloudscraper
```

그 다음 가상 환경이 활성화되어 있는지 확인합니다. Windows에서는 다음을 실행합니다:

```sh
.<venv-name>\Scripts\activate.bat
```

Linux 또는 macOS에서는 다음을 실행합니다:

```sh
source <venv-name>/bin/activate
```

### `cloudscraper can’t bypass the latest Cloudflare version`

패키지를 업데이트합니다:

```sh
pip install -U cloudscraper
```

## cloudscraper Alternatives

Bright Data는 Cloudflare를 우회하기 위한 견고한 프록시 네트워크를 제공합니다. 계정을 생성하고 설정한 다음 API 자격 증명을 발급받습니다. 그 다음 해당 자격 증명을 사용하여 아래와 같이 대상 URL의 데이터에 접근합니다:

```python
import requests

host = 'brd.superproxy.io'
port = 22225

username = 'brd-customer-<customer_id>-zone-<zone_name>'
password = '<zone_password>'

proxy_url = f'http://{username}:{password}@{host}:{port}'

proxies = {
    'http': proxy_url,
    'https': proxy_url
}

response = requests.get(URL, proxies=proxies)
```

여기서는 Python Requests 라이브러리로 `GET` 리クエ스트를 수행하고, `proxies` 매개변수로 프록시를 전달합니다.

## Conclusion

`cloudscraper`는 유용하지만 한계가 있습니다. Cloudflare로 보호되는 사이트에 접근하려면 Bright Data 프록시 네트워크와 [Web Unlocker](https://brightdata.co.kr/products/web-unlocker)를 사용해 보시는 것을 고려하시기 바랍니다.

지금 바로 무료 체험을 시작해 보시기 바랍니다!