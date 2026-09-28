import streamlit as st
import requests
import pandas as pd
import plotly.express as px
import re
from datetime import datetime

# --------------------------------------------------
# 기본 설정
# --------------------------------------------------
st.set_page_config(
    page_title="송탄고등학교 급식 영양 균형 분석",
    page_icon="🍚",
    layout="wide"
)

st.title("🍚 송탄고등학교 급식 영양 균형 분석")
st.subheader("탄수화물, 단백질, 지방이 가장 균형적으로 잡힌 날은 한 달에 얼마나 있을까?")

st.write(
    "송탄고등학교의 월별 중식 데이터를 분석하여 "
    "탄수화물·단백질·지방의 비율이 균형적인 날을 찾아봅니다."
)

# --------------------------------------------------
# API 인증키
# --------------------------------------------------
try:
    API_KEY = st.secrets["NEIS_KEY"]
except Exception:
    st.error(
        "NEIS_KEY가 설정되지 않았습니다. "
        "Streamlit Cloud의 Secrets에 NEIS_KEY를 입력해주세요."
    )
    st.stop()

SCHOOL_NAME = "송탄고등학교"

SCHOOL_INFO_URL = "https://open.neis.go.kr/hub/schoolInfo"
MEAL_URL = "https://open.neis.go.kr/hub/mealServiceDietInfo"


# --------------------------------------------------
# 학교 정보 가져오기
# --------------------------------------------------
@st.cache_data(ttl=86400)
def get_school_info():
    params = {
        "KEY": API_KEY,
        "Type": "json",
        "pIndex": 1,
        "pSize": 5,
        "SCHUL_NM": SCHOOL_NAME
    }

    response = requests.get(
        SCHOOL_INFO_URL,
        params=params,
        timeout=10
    )
    response.raise_for_status()

    data = response.json()

    if "schoolInfo" not in data:
        return None

    try:
        rows = data["schoolInfo"][1]["row"]
    except (KeyError, IndexError):
        return None

    # 정확히 송탄고등학교인 학교 우선 선택
    exact = [
        row for row in rows
        if row.get("SCHUL_NM") == SCHOOL_NAME
    ]

    if exact:
        return exact[0]

    if rows:
        return rows[0]

    return None


school = get_school_info()

if school is None:
    st.error("송탄고등학교의 학교 정보를 찾을 수 없습니다.")
    st.stop()

ATPT_CODE = school["ATPT_OFCDC_SC_CODE"]
SCHOOL_CODE = school["SD_SCHUL_CODE"]

st.caption(
    f"학교: {school['SCHUL_NM']} | "
    f"지역: {school.get('LCTN_SC_NM', '-')}"
)


# --------------------------------------------------
# 월 선택
# --------------------------------------------------
today = datetime.now()
current_year = today.year
current_month = today.month

col1, col2 = st.columns(2)

with col1:
    year = st.number_input(
        "연도",
        min_value=2020,
        max_value=current_year,
        value=current_year,
        step=1
    )

with col2:
    month = st.selectbox(
        "분석할 월",
        range(1, 13),
        index=current_month - 1,
        format_func=lambda x: f"{x}월"
    )


# --------------------------------------------------
# 기준 비율
# --------------------------------------------------
st.markdown("### ⚖️ 균형 기준")

target_carb = 55
target_protein = 20
target_fat = 25

c1, c2, c3 = st.columns(3)

with c1:
    st.metric("탄수화물 기준", f"{target_carb}%")

with c2:
    st.metric("단백질 기준", f"{target_protein}%")

with c3:
    st.metric("지방 기준", f"{target_fat}%")

st.caption(
    "탄수화물 55%, 단백질 20%, 지방 25%를 기준으로 "
    "실제 식단의 에너지 비율이 얼마나 가까운지 계산합니다."
)


# --------------------------------------------------
# 급식 데이터 가져오기
# --------------------------------------------------
@st.cache_data(ttl=3600)
def get_meal_data(year, month):
    start_date = f"{year}{month:02d}01"

    if month == 12:
        end_date = f"{year}1231"
    else:
        next_month = month + 1
        # 다음 달 1일에서 하루 빼기
        next_month_date = pd.Timestamp(
            year=year,
            month=next_month,
            day=1
        )
        last_day = next_month_date - pd.Timedelta(days=1)
        end_date = last_day.strftime("%Y%m%d")

    params = {
        "KEY": API_KEY,
        "Type": "json",
        "pIndex": 1,
        "pSize": 1000,
        "ATPT_OFCDC_SC_CODE": ATPT_CODE,
        "SD_SCHUL_CODE": SCHOOL_CODE,
        "MMEAL_SC_CODE": "2",
        "MLSV_FROM_YMD": start_date,
        "MLSV_TO_YMD": end_date
    }

    response = requests.get(
        MEAL_URL,
        params=params,
        timeout=15
    )
    response.raise_for_status()

    data = response.json()

    if "mealServiceDietInfo" not in data:
        return []

    try:
        rows = data["mealServiceDietInfo"][1]["row"]
    except (KeyError, IndexError):
        return []

    return rows


rows = get_meal_data(year, month)

if not rows:
    st.warning(
        f"{year}년 {month}월에는 송탄고등학교의 중식 급식 데이터가 없습니다."
    )
    st.stop()


# --------------------------------------------------
# NTR_INFO에서 탄수화물 / 단백질 / 지방 추출
# --------------------------------------------------
def extract_nutrition(ntr_info):
    if not ntr_info:
        return None, None, None

    carb = None
    protein = None
    fat = None

    # 예:
    # 탄수화물(g) : 100.4
    # 단백질(g) : 48.5
    # 지방(g) : 11.6

    carb_match = re.search(
        r"탄수화물\(g\)\s*:\s*([0-9.]+)",
        ntr_info
    )

    protein_match = re.search(
        r"단백질\(g\)\s*:\s*([0-9.]+)",
        ntr_info
    )

    fat_match = re.search(
        r"지방\(g\)\s*:\s*([0-9.]+)",
        ntr_info
    )

    if carb_match:
        carb = float(carb_match.group(1))

    if protein_match:
        protein = float(protein_match.group(1))

    if fat_match:
        fat = float(fat_match.group(1))

    return carb, protein, fat


# --------------------------------------------------
# 데이터 정리
# --------------------------------------------------
data = []

for row in rows:
    carb, protein, fat = extract_nutrition(
        row.get("NTR_INFO", "")
    )

    if carb is None or protein is None or fat is None:
        continue

    # 영양소별 에너지 계산
    carb_kcal = carb * 4
    protein_kcal = protein * 4
    fat_kcal = fat * 9

    total_kcal = carb_kcal + protein_kcal + fat_kcal

    if total_kcal == 0:
        continue

    carb_ratio = carb_kcal / total_kcal * 100
    protein_ratio = protein_kcal / total_kcal * 100
    fat_ratio = fat_kcal / total_kcal * 100

    # 기준과 실제 비율의 차이
    carb_diff = abs(carb_ratio - target_carb)
    protein_diff = abs(protein_ratio - target_protein)
    fat_diff = abs(fat_ratio - target_fat)

    total_diff = carb_diff + protein_diff + fat_diff

    # 차이가 0이면 100점
    # 차이가 커질수록 점수 감소
    balance_score = max(0, 100 - total_diff)

    data.append({
        "날짜": row["MLSV_YMD"],
        "탄수화물(g)": carb,
        "단백질(g)": protein,
        "지방(g)": fat,
        "탄수화물 비율": carb_ratio,
        "단백질 비율": protein_ratio,
        "지방 비율": fat_ratio,
        "총 에너지(kcal)": total_kcal,
        "균형 점수": balance_score,
        "메뉴": row.get("DDISH_NM", "")
    })


df = pd.DataFrame(data)

if df.empty:
    st.error("탄수화물·단백질·지방 정보가 있는 급식 데이터를 찾지 못했습니다.")
    st.stop()


# 날짜 변환
df["날짜"] = pd.to_datetime(
    df["날짜"],
    format="%Y%m%d"
)

df = df.sort_values("날짜").reset_index(drop=True)


# --------------------------------------------------
# 균형적인 날 판정
# --------------------------------------------------
# 기준에서 각 영양소의 차이가 총 10%p 이하인 날을
# '균형적인 날'로 정의
df["균형적인 날"] = df["균형 점수"] >= 90

balanced_df = df[df["균형적인 날"]].copy()

# 가장 균형적인 날
best_day = df.sort_values(
    "균형 점수",
    ascending=False
).iloc[0]


# --------------------------------------------------
# 결과 요약
# --------------------------------------------------
st.markdown("### 📊 분석 결과")

result1, result2, result3, result4 = st.columns(4)

with result1:
    st.metric(
        "전체 급식일",
        f"{len(df)}일"
    )

with result2:
    st.metric(
        "균형적인 날",
        f"{len(balanced_df)}일"
    )

with result3:
    if len(df) > 0:
        percentage = len(balanced_df) / len(df) * 100
    else:
        percentage = 0

    st.metric(
        "균형적인 날의 비율",
        f"{percentage:.1f}%"
    )

with result4:
    st.metric(
        "가장 균형적인 날",
        best_day["날짜"].strftime("%m월 %d일")
    )


# --------------------------------------------------
# 결과 해석
# --------------------------------------------------
st.markdown("### 🔎 결과 해석")

if len(balanced_df) == 0:
    st.info(
        f"{year}년 {month}월에는 기준에 맞는 균형적인 날이 "
        "없었습니다."
    )
else:
    st.success(
        f"{year}년 {month}월에는 총 {len(df)}번의 급식일 중 "
        f"{len(balanced_df)}일이 균형적인 날로 나타났습니다."
    )

st.write(
    f"가장 균형적인 날은 **{best_day['날짜'].strftime('%m월 %d일')}**이며, "
    f"균형 점수는 **{best_day['균형 점수']:.1f}점**입니다."
)


# --------------------------------------------------
# 그래프 1 : 날짜별 균형 점수
# --------------------------------------------------
st.markdown("### 📈 날짜별 균형 점수")

fig_score = px.bar(
    df,
    x="날짜",
    y="균형 점수",
    title="날짜별 탄수화물·단백질·지방 균형 점수",
    labels={
        "날짜": "날짜",
        "균형 점수": "균형 점수"
    },
    hover_data={
        "균형 점수": ":.1f"
    }
)

fig_score.add_hline(
    y=90,
    line_dash="dash",
    annotation_text="균형적인 날 기준 90점"
)

fig_score.update_yaxes(
    range=[
        max(0, min(70, df["균형 점수"].min() - 5)),
        100
    ]
)

st.plotly_chart(
    fig_score,
    use_container_width=True
)


# --------------------------------------------------
# 그래프 2 : 영양소 비율
# --------------------------------------------------
st.markdown("### 🥗 날짜별 탄수화물·단백질·지방 비율")

ratio_df = df[
    [
        "날짜",
        "탄수화물 비율",
        "단백질 비율",
        "지방 비율"
    ]
].copy()

ratio_long = ratio_df.melt(
    id_vars="날짜",
    var_name="영양소",
    value_name="비율"
)

fig_ratio = px.line(
    ratio_long,
    x="날짜",
    y="비율",
    color="영양소",
    markers=True,
    title="날짜별 3대 영양소 에너지 비율",
    labels={
        "날짜": "날짜",
        "비율": "에너지 비율 (%)",
        "영양소": "영양소"
    }
)

fig_ratio.add_hline(
    y=55,
    line_dash="dot",
    annotation_text="탄수화물 기준 55%"
)

fig_ratio.add_hline(
    y=20,
    line_dash="dot",
    annotation_text="단백질 기준 20%"
)

fig_ratio.add_hline(
    y=25,
    line_dash="dot",
    annotation_text="지방 기준 25%"
)

st.plotly_chart(
    fig_ratio,
    use_container_width=True
)


# --------------------------------------------------
# 가장 균형적인 날 상세 정보
# --------------------------------------------------
st.markdown("### 🏆 가장 균형적인 날")

best_col1, best_col2 = st.columns(2)

with best_col1:
    st.write(
        f"**날짜:** {best_day['날짜'].strftime('%Y년 %m월 %d일')}"
    )

    st.write(
        f"**균형 점수:** {best_day['균형 점수']:.1f}점"
    )

    st.write(
        f"**총 에너지:** {best_day['총 에너지(kcal)']:.1f} kcal"
    )

with best_col2:
    st.write(
        f"탄수화물: **{best_day['탄수화물(g)']:.1f}g** "
        f"({best_day['탄수화물 비율']:.1f}%)"
    )

    st.write(
        f"단백질: **{best_day['단백질(g)']:.1f}g** "
        f"({best_day['단백질 비율']:.1f}%)"
    )

    st.write(
        f"지방: **{best_day['지방(g)']:.1f}g** "
        f"({best_day['지방 비율']:.1f}%)"
    )


# --------------------------------------------------
# 균형적인 날 목록
# --------------------------------------------------
st.markdown("### 📅 균형적인 날 목록")

if balanced_df.empty:
    st.write("해당 월에는 균형적인 날이 없습니다.")
else:
    display_df = balanced_df[
        [
            "날짜",
            "탄수화물(g)",
            "단백질(g)",
            "지방(g)",
            "탄수화물 비율",
            "단백질 비율",
            "지방 비율",
            "균형 점수"
        ]
    ].copy()

    display_df["날짜"] = display_df["날짜"].dt.strftime(
        "%Y-%m-%d"
    )

    display_df = display_df.rename(
        columns={
            "탄수화물 비율": "탄수화물 비율(%)",
            "단백질 비율": "단백질 비율(%)",
            "지방 비율": "지방 비율(%)"
        }
    )

    st.dataframe(
        display_df.style.format({
            "탄수화물(g)": "{:.1f}",
            "단백질(g)": "{:.1f}",
            "지방(g)": "{:.1f}",
            "탄수화물 비율(%)": "{:.1f}",
            "단백질 비율(%)": "{:.1f}",
            "지방 비율(%)": "{:.1f}",
            "균형 점수": "{:.1f}"
        }),
        use_container_width=True,
        hide_index=True
    )


# --------------------------------------------------
# 전체 데이터
# --------------------------------------------------
with st.expander("📋 전체 급식 영양 데이터 보기"):
    all_display = df[
        [
            "날짜",
            "탄수화물(g)",
            "단백질(g)",
            "지방(g)",
            "탄수화물 비율",
            "단백질 비율",
            "지방 비율",
            "균형 점수"
        ]
    ].copy()

    all_display["날짜"] = all_display["날짜"].dt.strftime(
        "%Y-%m-%d"
    )

    all_display = all_display.rename(
        columns={
            "탄수화물 비율": "탄수화물 비율(%)",
            "단백질 비율": "단백질 비율(%)",
            "지방 비율": "지방 비율(%)"
        }
    )

    st.dataframe(
        all_display.style.format({
            "탄수화물(g)": "{:.1f}",
            "단백질(g)": "{:.1f}",
            "지방(g)": "{:.1f}",
            "탄수화물 비율(%)": "{:.1f}",
            "단백질 비율(%)": "{:.1f}",
            "지방 비율(%)": "{:.1f}",
            "균형 점수": "{:.1f}"
        }),
        use_container_width=True,
        hide_index=True
    )

st.caption(
    "데이터 출처: 나이스 교육정보 개방 포털 급식식단정보"
)
