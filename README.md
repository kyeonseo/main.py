import streamlit as st
import pandas as pd

# ------------------------------------------------------------
# 설정
# ------------------------------------------------------------
CSV_PATH = "seoul.csv"

st.set_page_config(
    page_title="서울 역대 기온 랭킹",
    page_icon="🌡️",
    layout="centered",
)

# ------------------------------------------------------------
# 데이터 로드 & 전처리
# ------------------------------------------------------------
@st.cache_data
def load_data(path: str) -> pd.DataFrame:
    df = pd.read_csv(path, encoding="utf-8-sig")
    df["날짜"] = df["날짜"].astype(str).str.strip().str.strip("\t").str.strip()
    df["날짜"] = pd.to_datetime(df["날짜"], errors="coerce")
    df = df.dropna(subset=["날짜", "평균기온"]).copy()
    df = df.sort_values("날짜").drop_duplicates(subset="날짜", keep="last")
    return df


df = load_data(CSV_PATH)

full_idx = pd.date_range(df["날짜"].min(), df["날짜"].max(), freq="D")
temp_series = df.set_index("날짜")["평균기온"].reindex(full_idx)

min_date = df["날짜"].min().date()
max_date = df["날짜"].max().date()

# ------------------------------------------------------------
# 스타일
# ------------------------------------------------------------
st.markdown(
    """
    <style>
    .main {background-color: #fafafa;}
    .title-wrap {text-align:center; padding: 6px 0 0 0;}
    .subtitle {text-align:center; color:#888; font-size:14px; margin-bottom: 28px;}
    .card {
        border-radius: 18px;
        padding: 22px 20px;
        text-align:center;
        box-shadow: 0 4px 16px rgba(0,0,0,0.08);
        height: 100%;
    }
    .hot-card {background: linear-gradient(135deg, #FFEDEB 0%, #FFD6CC 100%);}
    .cold-card {background: linear-gradient(135deg, #E8F4FF 0%, #D2E9FF 100%);}
    .card-label {font-size:15px; color:#555; font-weight:600; margin-bottom:6px;}
    .card-rank {font-size:34px; font-weight:800; margin:4px 0;}
    .hot-rank {color:#E4572E;}
    .cold-rank {color:#1E6FE0;}
    .card-sub {font-size:13px; color:#777;}
    .period-box {
        background:white; border-radius:14px; padding:16px 18px;
        box-shadow: 0 2px 10px rgba(0,0,0,0.06); margin-bottom:18px;
        text-align:center;
    }
    .bar-wrap {margin: 22px 4px 6px 4px;}
    .bar-bg {
        position:relative; height:14px; border-radius:8px;
        background: linear-gradient(90deg, #4A90E2 0%, #F2F2F2 50%, #E4572E 100%);
    }
    .bar-marker {
        position:absolute; top:-7px; width:3px; height:28px;
        background:#222; border-radius:2px;
    }
    .bar-caption {display:flex; justify-content:space-between; font-size:12px; color:#999; margin-top:4px;}
    .medal {font-size:22px;}
    </style>
    """,
    unsafe_allow_html=True,
)

st.markdown("<h1 class='title-wrap'>🌡️ 서울 역대 기온 랭킹</h1>", unsafe_allow_html=True)
st.markdown(
    f"<div class='subtitle'>기상 관측 데이터 {min_date} ~ {max_date} 기준</div>",
    unsafe_allow_html=True,
)

# ------------------------------------------------------------
# 기간 선택 (달력)
# ------------------------------------------------------------
default_end = max_date
default_start = default_end - pd.Timedelta(days=6)
if default_start < min_date:
    default_start = min_date

date_range = st.date_input(
    "📅 궁금한 기간을 달력에서 선택하세요",
    value=(default_start, default_end),
    min_value=min_date,
    max_value=max_date,
)

if not (isinstance(date_range, (tuple, list)) and len(date_range) == 2):
    st.info("시작일과 종료일을 모두 선택해주세요.")
    st.stop()

start_date, end_date = date_range
if start_date > end_date:
    st.error("시작일이 종료일보다 늦을 수 없어요!")
    st.stop()

start_ts = pd.Timestamp(start_date)
end_ts = pd.Timestamp(end_date)
n_days = (end_ts - start_ts).days + 1

period_vals = temp_series.loc[start_ts:end_ts]

if len(period_vals) < n_days or period_vals.isna().any():
    st.warning("⚠️ 선택한 기간에 관측 결측치가 있어 정확한 순위를 계산할 수 없어요. 다른 기간을 선택해주세요.")
    st.stop()

selected_mean = period_vals.mean()

st.markdown(
    f"""
    <div class="period-box">
        <div style="font-size:14px;color:#888;">선택한 기간 ({n_days}일)</div>
        <div style="font-size:20px;font-weight:700;margin-top:2px;">
            {start_ts.date()} ~ {end_ts.date()}
        </div>
        <div style="font-size:15px;color:#555;margin-top:6px;">
            평균기온 <b>{selected_mean:.1f}℃</b>
        </div>
    </div>
    """,
    unsafe_allow_html=True,
)

# ------------------------------------------------------------
# 동일 길이(n_days) 기간 전체에 대한 순위 계산
# ------------------------------------------------------------
rolling_mean = temp_series.rolling(window=n_days, min_periods=n_days).mean()
valid = rolling_mean.dropna()
total = len(valid)

hot_rank = int((valid > selected_mean).sum()) + 1
cold_rank = int((valid < selected_mean).sum()) + 1
hot_pct = hot_rank / total * 100
cold_pct = cold_rank / total * 100
percentile = (valid < selected_mean).sum() / total * 100  # 0=가장추움, 100=가장더움


def medal(rank: int) -> str:
    return {1: "🥇", 2: "🥈", 3: "🥉"}.get(rank, "")


col1, col2 = st.columns(2)
with col1:
    st.markdown(
        f"""
        <div class="card hot-card">
            <div class="card-label">🔥 더위 순위</div>
            <div class="card-rank hot-rank">{medal(hot_rank)} {hot_rank:,}위</div>
            <div class="card-sub">전체 {total:,}개 동일기간 중<br>상위 {hot_pct:.2f}%</div>
        </div>
        """,
        unsafe_allow_html=True,
    )
with col2:
    st.markdown(
        f"""
        <div class="card cold-card">
            <div class="card-label">❄️ 추위 순위</div>
            <div class="card-rank cold-rank">{medal(cold_rank)} {cold_rank:,}위</div>
            <div class="card-sub">전체 {total:,}개 동일기간 중<br>상위 {cold_pct:.2f}%</div>
        </div>
        """,
        unsafe_allow_html=True,
    )

# 퍼센타일 바
st.markdown(
    f"""
    <div class="bar-wrap">
        <div class="bar-bg">
            <div class="bar-marker" style="left: calc({percentile:.4f}% - 1.5px);"></div>
        </div>
        <div class="bar-caption">
            <span>❄️ 역대 가장 추운 기간</span>
            <span>🔥 역대 가장 더운 기간</span>
        </div>
    </div>
    """,
    unsafe_allow_html=True,
)

# 한 줄 총평
if hot_rank <= 3:
    st.success(f"🏆 역대 **{hot_rank}위**! 손에 꼽히는 폭염 기간이에요.")
elif cold_rank <= 3:
    st.info(f"🧊 역대 **{cold_rank}위**! 손에 꼽히는 한파 기간이에요.")
elif hot_pct <= 1:
    st.success("역대급 더위, 상위 1% 안에 드는 기간이에요 🔥")
elif cold_pct <= 1:
    st.info("역대급 추위, 상위 1% 안에 드는 기간이에요 🧊")
else:
    st.write(f"평범한 편에 속하는 기간이에요. (더위 {hot_pct:.1f}% / 추위 {cold_pct:.1f}%)")

# ------------------------------------------------------------
# Top 5 참고 테이블
# ------------------------------------------------------------
st.markdown("---")
t1, t2 = st.columns(2)


def make_table(series: pd.Series) -> pd.DataFrame:
    rows = []
    for end_d, v in series.items():
        s_d = end_d - pd.Timedelta(days=n_days - 1)
        rows.append(
            {
                "기간": f"{s_d.date()} ~ {end_d.date()}",
                "평균기온(℃)": round(float(v), 1),
            }
        )
    return pd.DataFrame(rows)


with t1:
    st.markdown(f"**🔥 역대 최고 더위 TOP 5** ({n_days}일 기준)")
    st.dataframe(make_table(valid.nlargest(5)), hide_index=True, use_container_width=True)

with t2:
    st.markdown(f"**❄️ 역대 최고 추위 TOP 5** ({n_days}일 기준)")
    st.dataframe(make_table(valid.nsmallest(5)), hide_index=True, use_container_width=True)

st.caption("※ 순위는 선택한 기간과 동일한 일수(n일)의 이동평균 기온을 서울 전체 관측기간과 비교하여 계산합니다.")
