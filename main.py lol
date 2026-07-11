from pathlib import Path

import pandas as pd
import plotly.express as px
import plotly.graph_objects as go
import streamlit as st


# =========================================================
# 기본 설정
# =========================================================

st.set_page_config(
    page_title="LoL 챔피언 분석실",
    page_icon="⚔️",
    layout="wide",
)

DATA_FILE = Path(__file__).with_name("LoL_챔피언.csv")


ROLE_KR = {
    "Fighter": "전사",
    "Mage": "마법사",
    "Marksman": "원거리 딜러",
    "Tank": "탱커",
    "Support": "서포터",
    "Assassin": "암살자",
}


STAT_COLUMNS = [
    "체력",
    "체력성장",
    "마나",
    "마나성장",
    "이동속도",
    "방어력",
    "마법저항력",
    "공격사거리",
    "공격력",
    "공격력성장",
    "공격속도성장(%)",
]


# =========================================================
# 화면 디자인
# =========================================================

st.markdown(
    """
    <style>
    .block-container {
        padding-top: 1.2rem;
        padding-bottom: 2rem;
    }

    .main-title {
        font-size: 2.3rem;
        font-weight: 800;
        margin-bottom: 0.1rem;
    }

    .sub-title {
        color: #8a8f98;
        margin-bottom: 1.2rem;
    }

    div[data-testid="stMetric"] {
        border: 1px solid rgba(128, 128, 128, 0.25);
        border-radius: 14px;
        padding: 12px;
    }
    </style>
    """,
    unsafe_allow_html=True,
)


# =========================================================
# 데이터 불러오기
# =========================================================

@st.cache_data
def load_data(path: Path) -> pd.DataFrame:
    if not path.exists():
        raise FileNotFoundError(
            f"{path.name} 파일을 app.py와 같은 폴더에 넣어 주세요."
        )

    data = pd.read_csv(path)

    required_columns = [
        "챔피언",
        "영문이름",
        "역할1",
        "역할2",
    ]

    missing_columns = [
        column
        for column in required_columns
        if column not in data.columns
    ]

    if missing_columns:
        raise ValueError(
            "CSV에 필요한 열이 없습니다: "
            + ", ".join(missing_columns)
        )

    data["역할2"] = data["역할2"].fillna("없음")

    data["역할1_한글"] = (
        data["역할1"]
        .map(ROLE_KR)
        .fillna(data["역할1"])
    )

    data["역할2_한글"] = (
        data["역할2"]
        .map(ROLE_KR)
        .fillna(data["역할2"])
    )

    return data


# =========================================================
# 백분위 계산
# =========================================================

def percentile_score(
    series: pd.Series,
    value: float,
) -> float:
    return float((series <= value).mean() * 100)


# =========================================================
# 레이더 차트 생성
# =========================================================

def make_radar_chart(
    data: pd.DataFrame,
    selected_names: list[str],
    selected_stats: list[str],
) -> go.Figure:

    figure = go.Figure()

    for champion_name in selected_names:
        champion_row = data.loc[
            data["챔피언"] == champion_name
        ].iloc[0]

        values = []

        for stat in selected_stats:
            stat_series = data[stat]

            min_value = float(stat_series.min())
            max_value = float(stat_series.max())

            if max_value == min_value:
                normalized_value = 50
            else:
                normalized_value = (
                    (
                        float(champion_row[stat])
                        - min_value
                    )
                    / (
                        max_value
                        - min_value
                    )
                    * 100
                )

            values.append(normalized_value)

        figure.add_trace(
            go.Scatterpolar(
                r=values + [values[0]],
                theta=selected_stats + [selected_stats[0]],
                fill="toself",
                name=champion_name,
            )
        )

    figure.update_layout(
        polar={
            "radialaxis": {
                "visible": True,
                "range": [0, 100],
            }
        },
        margin={
            "l": 35,
            "r": 35,
            "t": 45,
            "b": 35,
        },
        height=520,
        legend={
            "orientation": "h",
        },
    )

    return figure


# =========================================================
# 데이터 준비
# =========================================================

try:
    df = load_data(DATA_FILE)

except Exception as error:
    st.error(str(error))
    st.stop()


valid_stats = [
    column
    for column in STAT_COLUMNS
    if column in df.columns
    and df[column].nunique(dropna=True) > 1
]


# =========================================================
# 제목
# =========================================================

st.markdown(
    '<div class="main-title">⚔️ LoL 챔피언 분석실</div>',
    unsafe_allow_html=True,
)

st.markdown(
    """
    <div class="sub-title">
    챔피언 검색, 역할별 분석, 능력치 순위와 챔피언 비교
    </div>
    """,
    unsafe_allow_html=True,
)


# =========================================================
# 사이드바 필터
# =========================================================

with st.sidebar:

    st.header("분석 조건")

    search_keyword = st.text_input(
        "챔피언 검색",
        placeholder="예: 가렌 또는 Garen",
    )

    role_options = sorted(
        df["역할1_한글"].dropna().unique()
    )

    selected_roles = st.multiselect(
        "주 역할",
        options=role_options,
        default=role_options,
    )

    range_stat = st.selectbox(
        "범위 필터 능력치",
        valid_stats,
    )

    min_stat_value = float(df[range_stat].min())
    max_stat_value = float(df[range_stat].max())

    selected_range = st.slider(
        f"{range_stat} 범위",
        min_value=min_stat_value,
        max_value=max_stat_value,
        value=(
            min_stat_value,
            max_stat_value,
        ),
    )

    st.caption(
        "CSV 파일을 수정하면 사이트의 데이터도 변경됩니다."
    )


# =========================================================
# 필터 적용
# =========================================================

filtered_df = df[
    df["역할1_한글"].isin(selected_roles)
    & df[range_stat].between(
        selected_range[0],
        selected_range[1],
    )
].copy()


if search_keyword.strip():
    keyword = search_keyword.strip().lower()

    filtered_df = filtered_df[
        filtered_df["챔피언"]
        .str.lower()
        .str.contains(keyword, na=False)
        |
        filtered_df["영문이름"]
        .str.lower()
        .str.contains(keyword, na=False)
    ]


# =========================================================
# 탭 구성
# =========================================================

tab1, tab2, tab3, tab4 = st.tabs(
    [
        "📊 전체 현황",
        "🏆 능력치 순위",
        "⚔️ 챔피언 비교",
        "📋 데이터 보기",
    ]
)


# =========================================================
# 탭 1: 전체 현황
# =========================================================

with tab1:

    metric1, metric2, metric3, metric4 = st.columns(4)

    metric1.metric(
        "전체 챔피언",
        f"{len(df)}명",
    )

    metric2.metric(
        "현재 검색 결과",
        f"{len(filtered_df)}명",
    )

    metric3.metric(
        "주 역할 종류",
        f"{df['역할1'].nunique()}개",
    )

    if len(filtered_df) > 0:
        average_speed = filtered_df["이동속도"].mean()
        metric4.metric(
            "평균 이동속도",
            f"{average_speed:.1f}",
        )
    else:
        metric4.metric(
            "평균 이동속도",
            "-",
        )

    left_column, right_column = st.columns(2)

    with left_column:

        role_count = (
            filtered_df["역할1_한글"]
            .value_counts()
            .rename_axis("역할")
            .reset_index(name="챔피언 수")
        )

        role_chart = px.bar(
            role_count,
            x="역할",
            y="챔피언 수",
            text_auto=True,
            title="주 역할별 챔피언 수",
        )

        st.plotly_chart(
            role_chart,
            use_container_width=True,
        )

    with right_column:

        default_x_index = (
            valid_stats.index("공격력")
            if "공격력" in valid_stats
            else 0
        )

        default_y_index = (
            valid_stats.index("체력")
            if "체력" in valid_stats
            else 0
        )

        x_stat = st.selectbox(
            "가로축 능력치",
            valid_stats,
            index=default_x_index,
        )

        y_stat = st.selectbox(
            "세로축 능력치",
            valid_stats,
            index=default_y_index,
        )

        scatter_chart = px.scatter(
            filtered_df,
            x=x_stat,
            y=y_stat,
            color="역할1_한글",
            hover_name="챔피언",
            hover_data=[
                "영문이름",
                "역할2_한글",
            ],
            title=f"{x_stat}와 {y_stat}의 관계",
        )

        st.plotly_chart(
            scatter_chart,
            use_container_width=True,
        )

    st.subheader("역할별 평균 능력치")

    if len(filtered_df) > 0:

        role_mean_df = (
            filtered_df
            .groupby("역할1_한글")[valid_stats]
            .mean()
            .round(2)
            .reset_index()
        )

        st.dataframe(
            role_mean_df,
            use_container_width=True,
            hide_index=True,
        )

    else:
        st.warning("조건에 맞는 챔피언이 없습니다.")


# =========================================================
# 탭 2: 능력치 순위
# =========================================================

with tab2:

    option_column, chart_column = st.columns(
        [1, 2]
    )

    with option_column:

        ranking_stat = st.selectbox(
            "순위를 볼 능력치",
            valid_stats,
            key="ranking_stat",
        )

        top_n = st.slider(
            "표시할 챔피언 수",
            min_value=5,
            max_value=30,
            value=10,
        )

        order_option = st.radio(
            "정렬 방식",
            [
                "높은 순",
                "낮은 순",
            ],
            horizontal=True,
        )

    ascending = order_option == "낮은 순"

    ranking_df = (
        filtered_df
        .sort_values(
            ranking_stat,
            ascending=ascending,
        )
        .head(top_n)
    )

    with chart_column:

        ranking_chart = px.bar(
            ranking_df.sort_values(
                ranking_stat,
                ascending=not ascending,
            ),
            x=ranking_stat,
            y="챔피언",
            orientation="h",
            text=ranking_stat,
            color="역할1_한글",
            title=(
                f"{ranking_stat} "
                f"{order_option} TOP {top_n}"
            ),
        )

        st.plotly_chart(
            ranking_chart,
            use_container_width=True,
        )

    ranking_table = ranking_df[
        [
            "챔피언",
            "영문이름",
            "역할1_한글",
            "역할2_한글",
            ranking_stat,
        ]
    ].copy()

    ranking_table.insert(
        0,
        "순위",
        range(
            1,
            len(ranking_table) + 1,
        ),
    )

    st.dataframe(
        ranking_table,
        use_container_width=True,
        hide_index=True,
    )


# =========================================================
# 탭 3: 챔피언 비교
# =========================================================

with tab3:

    champion_names = df["챔피언"].tolist()

    default_compare_names = champion_names[:2]

    compare_names = st.multiselect(
        "비교할 챔피언을 2명에서 5명까지 선택하세요",
        options=champion_names,
        default=default_compare_names,
        max_selections=5,
    )

    default_radar_stats = [
        stat
        for stat in [
            "체력",
            "이동속도",
            "방어력",
            "마법저항력",
            "공격사거리",
            "공격력",
        ]
        if stat in valid_stats
    ]

    radar_stats = st.multiselect(
        "레이더 차트 능력치",
        options=valid_stats,
        default=default_radar_stats,
        max_selections=8,
    )

    if (
        len(compare_names) >= 2
        and len(radar_stats) >= 3
    ):

        compare_df = df[
            df["챔피언"].isin(compare_names)
        ].copy()

        radar_chart = make_radar_chart(
            df,
            compare_names,
            radar_stats,
        )

        st.plotly_chart(
            radar_chart,
            use_container_width=True,
        )

        compare_columns = [
            "챔피언",
            "영문이름",
            "역할1_한글",
            "역할2_한글",
        ] + radar_stats

        st.dataframe(
            compare_df[compare_columns],
            use_container_width=True,
            hide_index=True,
        )

        st.subheader(
            "전체 챔피언 중 능력치 위치"
        )

        selected_champion = st.selectbox(
            "백분위로 볼 챔피언",
            compare_names,
        )

        selected_row = df[
            df["챔피언"]
            == selected_champion
        ].iloc[0]

        metric_count = min(
            4,
            len(radar_stats),
        )

        metric_columns = st.columns(
            metric_count
        )

        for index, stat in enumerate(
            radar_stats
        ):

            score = percentile_score(
                df[stat],
                selected_row[stat],
            )

            upper_percent = 100 - score

            metric_columns[
                index % metric_count
            ].metric(
                stat,
                f"{selected_row[stat]:g}",
                f"상위 약 {upper_percent:.1f}%",
            )

    else:
        st.info(
            "챔피언 2명 이상과 능력치 3개 이상을 선택해 주세요."
        )


# =========================================================
# 탭 4: 데이터 보기
# =========================================================

with tab4:

    display_columns = [
        "챔피언",
        "영문이름",
        "역할1_한글",
        "역할2_한글",
    ] + valid_stats

    sort_column = st.selectbox(
        "정렬 기준",
        display_columns,
        index=0,
    )

    sort_ascending = st.checkbox(
        "오름차순",
        value=True,
    )

    display_df = (
        filtered_df[display_columns]
        .sort_values(
            sort_column,
            ascending=sort_ascending,
        )
    )

    st.dataframe(
        display_df,
        use_container_width=True,
        hide_index=True,
    )

    csv_data = (
        display_df
        .to_csv(index=False)
        .encode("utf-8-sig")
    )

    st.download_button(
        label="필터 결과 CSV 다운로드",
        data=csv_data,
        file_name="LoL_챔피언_분석결과.csv",
        mime="text/csv",
    )


# =========================================================
# 데이터 안내
# =========================================================

with st.expander("데이터 품질 안내"):

    st.write(
        """
- 역할2가 비어 있는 챔피언은 `없음`으로 표시합니다.
- 모든 값이 같은 능력치는 그래프 선택 항목에서 제외합니다.
- 현재 데이터에서 공격력성장 값이 모두 0이면 자동으로 제외됩니다.
- 마나가 없는 챔피언이나 특수 자원을 사용하는 챔피언은 마나 값이 0일 수 있습니다.
        """
    )
