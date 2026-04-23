# KANGMA
furniture-crawler

import io
import json
import re
from typing import Any
from urllib.parse import urljoin, urlparse

import pandas as pd
import requests
import streamlit as st
from bs4 import BeautifulSoup


st.set_page_config(page_title="가구 상세페이지 수집기", layout="wide")

COLUMNS = [
    "source_url",
    "thumbnail_url",
    "brand_name",
    "product_code",
    "product_name",
    "category",
    "size",
    "color",
    "price_original",
    "material",
    "shipping_type",
    "shipping_fee",
    "origin",
    "ai_name_ko",
    "ai_name_en",
    "ai_features",
    "price_calc",
    "special_notes",
    "status",
]

HEADERS = {
    "User-Agent": (
        "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
        "AppleWebKit/537.36 (KHTML, like Gecko) "
        "Chrome/123.0.0.0 Safari/537.36"
    )
}

DEFAULT_RULES = [
    {
        "name": "세라믹 소재",
        "keywords": ["세라믹", "ceramic"],
        "message": "패턴/색상 랜덤 발송 안내",
    },
    {
        "name": "미조립 가구",
        "keywords": ["미조립", "조립 필요", "knock down"],
        "message": "미조립 제품 안내",
    },
    {
        "name": "대형 가구",
        "keywords": ["대형", "large furniture"],
        "message": "택배 배송 불가",
    },
    {
        "name": "나무 의자",
        "keywords": ["나무 의자", "우드 체어", "wood chair"],
        "message": "택배 배송 불가",
    },
    {
        "name": "국내 생산",
        "keywords": ["국내 생산", "국내 제작", "made in korea"],
        "message": "국내 제작 안내",
    },
    {
        "name": "주문제작 생산",
        "keywords": ["주문제작", "made to order"],
        "message": "주문제작기간 안내",
    },
    {
        "name": "중국 외 생산",
        "keywords": [
            "이탈리아",
            "베트남",
            "터키",
            "폴란드",
            "말레이시아",
            "인도네시아",
            "italy",
            "vietnam",
            "turkey",
            "poland",
            "malaysia",
            "indonesia",
        ],
        "message": "원산지 강조",
    },
]

CATEGORY_KEYWORDS = [
    "의자",
    "체어",
    "chair",
    "소파",
    "sofa",
    "테이블",
    "table",
    "책상",
    "desk",
    "스툴",
    "stool",
    "벤치",
    "bench",
    "수납장",
    "cabinet",
    "서랍장",
    "dresser",
    "침대",
    "bed",
    "협탁",
    "side table",
    "콘솔",
    "console",
]


if "items" not in st.session_state:
    st.session_state.items = []

if "url_input" not in st.session_state:
    st.session_state.url_input = ""


def load_rules() -> list[dict[str, Any]]:
    try:
        with open("rules.json", "r", encoding="utf-8") as f:
            data = json.load(f)
            if isinstance(data, list):
                return data
    except Exception:
        pass
    return DEFAULT_RULES


def clean_text(value: Any) -> str:
    if value is None:
        return ""
    text = str(value).replace("\xa0", " ")
    text = re.sub(r"\s+", " ", text)
    return text.strip()


def fetch_html(url: str) -> str:
    response = requests.get(url, headers=HEADERS, timeout=20)
    response.raise_for_status()
    return response.text


def safe_json_load(raw: str) -> Any:
    try:
        return json.loads(raw)
    except Exception:
        return None


def find_meta_content(soup: BeautifulSoup, name_or_property: str) -> str:
    tag = soup.find("meta", attrs={"property": name_or_property})
    if not tag:
        tag = soup.find("meta", attrs={"name": name_or_property})
    if tag and tag.get("content"):
        return clean_text(tag.get("content"))
    return ""


def extract_jsonld_product(soup: BeautifulSoup) -> dict[str, Any]:
    scripts = soup.find_all("script", type="application/ld+json")
    for script in scripts:
        raw = script.string or script.get_text(strip=True)
        if not raw:
            continue

        data = safe_json_load(raw)
        if data is None:
            continue

        nodes = []
        if isinstance(data, list):
            nodes = data
        elif isinstance(data, dict):
            if "@graph" in data and isinstance(data["@graph"], list):
                nodes = data["@graph"]
            else:
                nodes = [data]

        for node in nodes:
            if not isinstance(node, dict):
                continue
            node_type = node.get("@type")
            if node_type == "Product":
                return node
            if isinstance(node_type, list) and "Product" in node_type:
                return node
    return {}


def extract_brand_name(soup: BeautifulSoup, page_text: str, url: str) -> str:
    candidates = []

    og_site = find_meta_content(soup, "og:site_name")
    if og_site:
        candidates.append(og_site)

    app_name = soup.find("meta", attrs={"name": "application-name"})
    if app_name and app_name.get("content"):
        candidates.append(clean_text(app_name["content"]))

    title = clean_text(soup.title.string if soup.title else "")
    if "|" in title:
        candidates.append(title.split("|")[-1].strip())
    if "-" in title:
        candidates.append(title.split("-")[-1].strip())

    for candidate in candidates:
        if candidate and len(candidate) <= 40:
            return candidate

    host = urlparse(url).netloc.replace("www.", "")
    return host.split(".")[0]


def extract_product_code(page_text: str) -> str:
    patterns = [
        r"(?:상품코드|모델명|품번|product\s*code|model|sku)\s*[:：]?\s*([A-Za-z0-9\-_]{3,})",
        r"(?:상품번호|품목코드)\s*[:：]?\s*([A-Za-z0-9\-_]{3,})",
    ]
    for pattern in patterns:
        match = re.search(pattern, page_text, re.IGNORECASE)
        if match:
            return clean_text(match.group(1))
    return ""


def extract_origin(page_text: str) -> str:
    patterns = [
        r"(?:원산지|제조국|생산지|country of origin)\s*[:：]?\s*([^\n\r|/]{2,30})",
    ]
    for pattern in patterns:
        match = re.search(pattern, page_text, re.IGNORECASE)
        if match:
            value = clean_text(match.group(1))
            value = re.sub(r"(상세|별도|참고).*", "", value).strip()
            return value
    return ""


def extract_price_candidates(text: str) -> list[str]:
    patterns = [
        r"([0-9]{1,3}(?:,[0-9]{3})+\s*원)",
        r"([0-9]+\s*원)",
    ]
    found: list[str] = []
    for pattern in patterns:
        found.extend(re.findall(pattern, text))

    unique: list[str] = []
    for item in found:
        item = clean_text(item)
        if item not in unique:
            unique.append(item)
    return unique


def normalize_price_to_number(price_text: str) -> int | None:
    only_num = re.sub(r"[^0-9]", "", price_text or "")
    return int(only_num) if only_num else None


def format_price(number: int | None) -> str:
    if number is None:
        return ""
    return f"{number:,}원"


def calc_sell_price(price: int | None) -> int | None:
    if not price:
        return None
    result = ((price * 0.5) * 1.4) * 1.1
    return round(result)


def extract_shipping_info(text: str) -> tuple[str, str]:
    shipping_type = ""
    shipping_fee = ""

    if "화물" in text:
        shipping_type = "화물배송"
    elif "택배" in text:
        shipping_type = "택배"

    fee_patterns = [
        r"배송비[^0-9]{0,10}([0-9]{1,3}(?:,[0-9]{3})*\s*원)",
        r"(?:택배비|화물비|운임)[^0-9]{0,10}([0-9]{1,3}(?:,[0-9]{3})*\s*원)",
    ]
    for pattern in fee_patterns:
        matches = re.findall(pattern, text)
        if matches:
            shipping_fee = clean_text(matches[0])
            break

    return shipping_type, shipping_fee


def normalize_size_tokens(text: str) -> str:
    return (
        text.replace("×", "x")
        .replace("X", "x")
        .replace("*", "x")
        .replace("＊", "x")
        .replace("–", "-")
        .replace("—", "-")
    )


def extract_size(text: str) -> str:
    src = normalize_size_tokens(text)

    chair_pattern = re.search(
        r"W\s*(\d+)\s*x\s*D\s*(\d+)\s*x\s*H\s*(\d+)\s*x\s*SH\s*(\d+)(?:\s*x\s*AH\s*(\d+))?",
        src,
        re.IGNORECASE,
    )
    if chair_pattern:
        groups = chair_pattern.groups()
        if groups[4]:
            return f"W{groups[0]} x D{groups[1]} x H{groups[2]} x SH{groups[3]} x AH{groups[4]}"
        return f"W{groups[0]} x D{groups[1]} x H{groups[2]} x SH{groups[3]}"

    basic_pattern = re.search(
        r"W\s*(\d+)\s*x\s*D\s*(\d+)\s*x\s*H\s*(\d+)",
        src,
        re.IGNORECASE,
    )
    if basic_pattern:
        g = basic_pattern.groups()
        return f"W{g[0]} x D{g[1]} x H{g[2]}"

    round_pattern = re.search(
        r"Ø\s*(\d+)\s*x\s*H\s*(\d+)",
        src,
        re.IGNORECASE,
    )
    if round_pattern:
        g = round_pattern.groups()
        return f"Ø{g[0]} x H{g[1]}"

    round_pattern_alt = re.search(
        r"(?:직경|지름)\s*(\d+)\s*(?:mm)?\s*x\s*H\s*(\d+)",
        src,
        re.IGNORECASE,
    )
    if round_pattern_alt:
        g = round_pattern_alt.groups()
        return f"Ø{g[0]} x H{g[1]}"

    top_pattern = re.search(
        r"W\s*(\d+)\s*x\s*D\s*(\d+)\s*x\s*(\d+)T\s*x\s*(\d+)R",
        src,
        re.IGNORECASE,
    )
    if top_pattern:
        g = top_pattern.groups()
        return f"W{g[0]} x D{g[1]} x {g[2]}T x {g[3]}R"

    simple_numeric = re.search(
        r"(\d{2,4})\s*x\s*(\d{2,4})\s*x\s*(\d{2,4})(?:\s*x\s*(\d{2,4}))?(?:\s*x\s*(\d{2,4}))?",
        src,
        re.IGNORECASE,
    )
    if simple_numeric:
        nums = simple_numeric.groups()
        if nums[3] and nums[4]:
            return f"W{nums[0]} x D{nums[1]} x H{nums[2]} x SH{nums[3]} x AH{nums[4]}"
        if nums[3]:
            return f"W{nums[0]} x D{nums[1]} x H{nums[2]} x SH{nums[3]}"
        return f"W{nums[0]} x D{nums[1]} x H{nums[2]}"

    return ""


def extract_category(soup: BeautifulSoup, page_text: str, product_name: str) -> str:
    breadcrumb_selectors = [
        "nav a",
        ".breadcrumb a",
        ".xans-product-headcategory a",
        ".path a",
        ".location a",
    ]
    for selector in breadcrumb_selectors:
        links = soup.select(selector)
        texts = [clean_text(a.get_text()) for a in links if clean_text(a.get_text())]
        if len(texts) >= 2:
            return " > ".join(texts)

    combined = f"{product_name} {page_text}".lower()
    for keyword in CATEGORY_KEYWORDS:
        if keyword.lower() in combined:
            return keyword.upper() if keyword.isascii() else keyword
    return ""


def extract_material(page_text: str) -> str:
    patterns = [
        r"(?:소재|재질|material)\s*[:：]?\s*([^\n\r]{2,120})",
    ]
    for pattern in patterns:
        match = re.search(pattern, page_text, re.IGNORECASE)
        if match:
            value = clean_text(match.group(1))
            return value[:120]
    return ""


def extract_color(page_text: str) -> str:
    patterns = [
        r"(?:색상|컬러|color)\s*[:：]?\s*([^\n\r]{2,120})",
    ]
    for pattern in patterns:
        match = re.search(pattern, page_text, re.IGNORECASE)
        if match:
            value = clean_text(match.group(1))
            return value[:120]
    return ""


def extract_product_name(soup: BeautifulSoup, jsonld: dict[str, Any]) -> str:
    candidates = []

    if jsonld.get("name"):
        candidates.append(clean_text(jsonld.get("name")))

    og_title = find_meta_content(soup, "og:title")
    if og_title:
        candidates.append(og_title)

    h1 = soup.find("h1")
    if h1:
        candidates.append(clean_text(h1.get_text()))

    title = clean_text(soup.title.string if soup.title else "")
    if title:
        candidates.append(title)

    for candidate in candidates:
        if candidate:
            return candidate
    return ""


def extract_thumbnail_url(soup: BeautifulSoup, jsonld: dict[str, Any], url: str) -> str:
    image_value = jsonld.get("image", "")
    if isinstance(image_value, list) and image_value:
        image_value = image_value[0]
    thumbnail = clean_text(image_value) or find_meta_content(soup, "og:image")

    if thumbnail and not thumbnail.startswith("http"):
        thumbnail = urljoin(url, thumbnail)
    return thumbnail


def extract_price(soup: BeautifulSoup, jsonld: dict[str, Any], page_text: str) -> str:
    offers = jsonld.get("offers", {})
    if isinstance(offers, list) and offers:
        offers = offers[0]

    if isinstance(offers, dict):
        price = offers.get("price")
        if price:
            raw = re.sub(r"[^0-9.]", "", str(price))
            if raw:
                try:
                    value = int(float(raw))
                    return format_price(value)
                except Exception:
                    pass

    price_candidates = extract_price_candidates(page_text)

    if not price_candidates:
        return ""

    def score_price(p: str) -> tuple[int, int]:
        num = normalize_price_to_number(p) or 0
        preferred = 0
        if 1000 <= num <= 100_000_000:
            preferred += 1
        return (preferred, num)

    price_candidates.sort(key=score_price, reverse=True)
    return price_candidates[0]


def split_product_name_for_generation(product_name: str, brand_name: str) -> str:
    name = clean_text(product_name)
    if brand_name:
        name = name.replace(brand_name, "").strip()
    name = re.sub(r"\[[^\]]+\]", "", name).strip()
    return clean_text(name)


def generate_similar_names(product_name: str, brand_name: str = "") -> tuple[str, str]:
    base = split_product_name_for_generation(product_name, brand_name)

    replacements_ko = {
        "루니": "루나",
        "모노": "모나",
        "코지": "코나",
        "엘린": "엘라",
        "로이": "로아",
        "노바": "노엘",
        "리브": "리벤",
        "마론": "마르",
    }
    replacements_en = {
        "LOONY": "LUNA",
        "MONO": "MONA",
        "COZY": "KONA",
        "ELIN": "ELLA",
        "ROY": "ROA",
        "NOVA": "NOEL",
        "LIVE": "LIVA",
    }

    ko_name = base
    for old, new in replacements_ko.items():
        if old in ko_name:
            ko_name = ko_name.replace(old, new)
            break

    if ko_name == base:
        if "의자" in base or "체어" in base:
            ko_name = "노바 의자"
        elif "테이블" in base:
            ko_name = "노바 테이블"
        elif "소파" in base:
            ko_name = "노바 소파"
        elif "스툴" in base:
            ko_name = "노바 스툴"
        else:
            ko_name = f"{base} N" if base else "노바"

    upper = base.upper()
    en_name = upper

    for old, new in replacements_en.items():
        if old in upper:
            en_name = upper.replace(old, new)
            break

    if en_name == upper:
        if "의자" in base or "체어" in base or "chair" in upper:
            en_name = "NOVA CHAIR"
        elif "테이블" in base or "table" in upper:
            en_name = "NOVA TABLE"
        elif "소파" in base or "sofa" in upper:
            en_name = "NOVA SOFA"
        elif "스툴" in base or "stool" in upper:
            en_name = "NOVA STOOL"
        else:
            en_name = "NOVA SERIES"

    return ko_name, en_name


def generate_features(
    product_name: str,
    material: str,
    color: str,
    size: str,
    shipping_type: str,
    category: str,
) -> str:
    line1 = (
        f"{product_name or '해당 제품'}은 공간에 안정적으로 어우러지는 "
        f"{category or '가구'} 타입의 디자인입니다."
    )

    if material:
        line2 = f"{material} 기준으로 질감과 분위기 연출 포인트를 잡기 좋은 제품입니다."
    elif color:
        line2 = f"{color} 정보를 바탕으로 공간 톤과의 매칭을 검토하기 좋습니다."
    else:
        line2 = "디테일 정보가 추가되면 소재감과 스타일 방향을 더 구체적으로 정리할 수 있습니다."

    if size and shipping_type:
        line3 = f"{size} / {shipping_type} 기준으로 배치 검토와 납품 조건 확인이 가능합니다."
    elif size:
        line3 = f"{size} 기준으로 주거 및 상업 공간 배치 검토가 가능합니다."
    elif shipping_type:
        line3 = f"{shipping_type} 기준으로 납품 방식과 설치 환경을 함께 검토할 수 있습니다."
    else:
        line3 = "사이즈와 배송 정보가 보강되면 실무용 상품 데이터로 바로 활용하기 좋습니다."

    return "\n".join([line1, line2, line3])


def detect_special_notes(
    product_name: str,
    material: str,
    shipping_type: str,
    origin: str,
    page_text: str,
    rules: list[dict[str, Any]],
) -> str:
    combined = " ".join(
        [
            clean_text(product_name),
            clean_text(material),
            clean_text(shipping_type),
            clean_text(origin),
            clean_text(page_text),
        ]
    ).lower()

    notes = []
    for rule in rules:
        keywords = [str(k).lower() for k in rule.get("keywords", [])]
        if any(keyword in combined for keyword in keywords):
            msg = clean_text(rule.get("message", ""))
            if msg:
                notes.append(msg)

    return " / ".join(sorted(set(notes)))


def parse_product(url: str, rules: list[dict[str, Any]]) -> dict[str, Any]:
    html = fetch_html(url)
    soup = BeautifulSoup(html, "lxml")
    page_text = clean_text(soup.get_text(" ", strip=True))
    jsonld = extract_jsonld_product(soup)

    brand_name = extract_brand_name(soup, page_text, url)
    product_code = extract_product_code(page_text)
    product_name = extract_product_name(soup, jsonld)
    thumbnail_url = extract_thumbnail_url(soup, jsonld, url)
    category = extract_category(soup, page_text, product_name)
    price_original = extract_price(soup, jsonld, page_text)
    material = extract_material(page_text)
    color = extract_color(page_text)
    size = extract_size(page_text)
    shipping_type, shipping_fee = extract_shipping_info(page_text)
    origin = extract_origin(page_text)

    ai_name_ko, ai_name_en = generate_similar_names(product_name, brand_name)
    ai_features = generate_features(
        product_name=product_name,
        material=material,
        color=color,
        size=size,
        shipping_type=shipping_type,
        category=category,
    )

    price_num = normalize_price_to_number(price_original)
    price_calc = calc_sell_price(price_num)
    price_calc_str = format_price(price_calc) if price_calc else ""

    special_notes = detect_special_notes(
        product_name=product_name,
        material=material,
        shipping_type=shipping_type,
        origin=origin,
        page_text=page_text,
        rules=rules,
    )

    return {
        "source_url": url,
        "thumbnail_url": thumbnail_url,
        "brand_name": brand_name,
        "product_code": product_code,
        "product_name": product_name,
        "category": category,
        "size": size,
        "color": color,
        "price_original": price_original,
        "material": material,
        "shipping_type": shipping_type,
        "shipping_fee": shipping_fee,
        "origin": origin,
        "ai_name_ko": ai_name_ko,
        "ai_name_en": ai_name_en,
        "ai_features": ai_features,
        "price_calc": price_calc_str,
        "special_notes": special_notes,
        "status": "parsed",
    }


def dataframe_from_items() -> pd.DataFrame:
    if not st.session_state.items:
        return pd.DataFrame(columns=COLUMNS)
    return pd.DataFrame(st.session_state.items, columns=COLUMNS)


def to_excel_bytes(df: pd.DataFrame) -> bytes:
    output = io.BytesIO()
    with pd.ExcelWriter(output, engine="openpyxl") as writer:
        df.to_excel(writer, index=False, sheet_name="products")
    output.seek(0)
    return output.getvalue()


def add_result(item: dict[str, Any]) -> None:
    st.session_state.items.append(item)


def render_result_cards(df: pd.DataFrame) -> None:
    if df.empty:
        st.info("아직 분석된 결과가 없습니다.")
        return

    st.markdown("### 결과 카드")
    for _, row in df.iterrows():
        with st.container(border=True):
            c1, c2 = st.columns([1, 5])
            with c1:
                thumb = clean_text(row.get("thumbnail_url", ""))
                if thumb:
                    st.image(thumb, width=110)
                else:
                    st.caption("썸네일 없음")

            with c2:
                st.markdown(
                    f"**{clean_text(row.get('product_name', '')) or '제품명 없음'}**"
                )
                st.caption(
                    f"브랜드: {clean_text(row.get('brand_name', '')) or '-'} | "
                    f"상품코드: {clean_text(row.get('product_code', '')) or '-'} | "
                    f"원산지: {clean_text(row.get('origin', '')) or '-'}"
                )
                st.caption(
                    f"카테고리: {clean_text(row.get('category', '')) or '-'} | "
                    f"사이즈: {clean_text(row.get('size', '')) or '-'} | "
                    f"색상: {clean_text(row.get('color', '')) or '-'}"
                )
                st.caption(
                    f"가격: {clean_text(row.get('price_original', '')) or '-'} | "
                    f"계산가: {clean_text(row.get('price_calc', '')) or '-'} | "
                    f"배송: {clean_text(row.get('shipping_type', '')) or '-'} "
                    f"{clean_text(row.get('shipping_fee', '')) or ''}"
                )
                if clean_text(row.get("special_notes", "")):
                    st.warning(clean_text(row.get("special_notes", "")))
                st.caption(f"상태: {clean_text(row.get('status', '')) or '-'}")


rules = load_rules()

st.title("가구 상세페이지 수집기")
st.caption("여러 상품 URL을 넣으면 공통 정보 추출 → 편집 → CSV/엑셀 내보내기까지 진행합니다.")

with st.expander("사용 안내", expanded=False):
    st.markdown(
        """
- 한 줄에 URL 하나씩 입력하세요.
- 사이트 구조가 제각각이라 일부 값은 비어 있을 수 있습니다.
- 추출 후 아래 표에서 직접 수정할 수 있습니다.
- 규칙 파일은 `rules.json`에서 나중에도 수정 가능합니다.
"""
    )

st.text_area(
    "상품 URL 입력",
    key="url_input",
    height=160,
    placeholder=(
        "https://example.com/product/1\n"
        "https://example.com/product/2\n"
        "https://example.com/product/3"
    ),
)

btn_col1, btn_col2, btn_col3 = st.columns([1, 1, 2])

with btn_col1:
    analyze_btn = st.button("분석 시작", use_container_width=True)

with btn_col2:
    clear_btn = st.button("전체 초기화", use_container_width=True)

if clear_btn:
    st.session_state.items = []
    st.session_state.url_input = ""
    st.rerun()

if analyze_btn:
    urls = [u.strip() for u in st.session_state.url_input.splitlines() if u.strip()]

    if not urls:
        st.warning("URL을 1개 이상 입력해주세요.")
    else:
        progress = st.progress(0)
        for idx, url in enumerate(urls, start=1):
            try:
                item = parse_product(url, rules)
                add_result(item)
            except Exception as e:
                add_result(
                    {
                        "source_url": url,
                        "thumbnail_url": "",
                        "brand_name": "",
                        "product_code": "",
                        "product_name": "",
                        "category": "",
                        "size": "",
                        "color": "",
                        "price_original": "",
                        "material": "",
                        "shipping_type": "",
                        "shipping_fee": "",
                        "origin": "",
                        "ai_name_ko": "",
                        "ai_name_en": "",
                        "ai_features": "",
                        "price_calc": "",
                        "special_notes": "",
                        "status": f"error: {clean_text(str(e))}",
                    }
                )
            progress.progress(idx / len(urls))
        st.success(f"{len(urls)}개 URL 분석이 완료되었습니다.")

df = dataframe_from_items()

render_result_cards(df)

st.markdown("### 편집 테이블")
edited_df = st.data_editor(
    df,
    use_container_width=True,
    hide_index=True,
    num_rows="dynamic",
    key="product_editor",
    column_config={
        "thumbnail_url": st.column_config.LinkColumn("thumbnail_url"),
        "source_url": st.column_config.LinkColumn("source_url"),
        "ai_features": st.column_config.TextColumn("ai_features", width="large"),
        "special_notes": st.column_config.TextColumn("special_notes", width="medium"),
        "status": st.column_config.TextColumn("status", width="medium"),
    },
)

if not edited_df.empty:
    st.session_state.items = edited_df.to_dict(orient="records")

export_df = edited_df.copy()

csv_data = export_df.to_csv(index=False).encode("utf-8-sig")
excel_data = to_excel_bytes(export_df)

down1, down2 = st.columns(2)
with down1:
    st.download_button(
        "CSV 다운로드",
        data=csv_data,
        file_name="furniture_products.csv",
        mime="text/csv",
        use_container_width=True,
    )

with down2:
    st.download_button(
        "엑셀 다운로드",
        data=excel_data,
        file_name="furniture_products.xlsx",
        mime="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
        use_container_width=True,
    )

