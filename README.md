# my-data-app
import streamlit as st
import pandas as pd
import requests
from datetime import datetime, timedelta
from zoneinfo import ZoneInfo

# ------------------------------------------------------------
# 1. 기본 설정
# ------------------------------------------------------------
# 화면 제목, 아이콘 등 페이지 전체 설정을 가장 먼저 해줍니다.
st.set_page_config(
    page_title="어제의 박스오피스",
    page_icon="🎬",
    layout="wide",
)

# KOBIS(영화진흥위원회) 일별 박스오피스 API 주소
API_URL = "https://www.kobis.or.kr/kobisopenapi/webservice/rest/boxoffice/searchDailyBoxOfficeList.json"


# ------------------------------------------------------------
# 2. '어제' 날짜를 한국 시간 기준으로 계산하기
# ------------------------------------------------------------
# 스트림릿 클라우드 서버는 한국 시간이 아닐 수 있으므로,
# zoneinfo를 이용해 "지금 한국은 몇 시인지"를 정확히 구한 뒤 하루를 뺍니다.
def get_yesterday_kst() -> str:
    now_kst = datetime.now(ZoneInfo("Asia/Seoul"))
    yesterday = now_kst - timedelta(days=1)
    # KOBIS API는 yyyymmdd 여덟 자리 형식을 요구합니다.
    return yesterday.strftime("%Y%m%d")


# ------------------------------------------------------------
# 3. API 호출 함수 (결과를 1시간 동안 기억해두는 캐시 적용)
# ------------------------------------------------------------
# ttl=3600 은 "이 함수의 결과를 3600초(1시간) 동안 재사용한다"는 뜻입니다.
# 같은 target_dt로 다시 호출되면, 실제로 API를 부르지 않고
# 캐시에 저장된 값을 그대로 돌려줍니다.
@st.cache_data(ttl=3600)
def fetch_box_office(target_dt: str):
    """
    KOBIS API를 호출해서 박스오피스 데이터를 가져옵니다.
    반환값은 (성공여부, 데이터 또는 에러메시지) 형태의 튜플입니다.
    """
    # 인증키는 코드에 직접 쓰지 않고, 스트림릿의 비밀 금고(secrets)에서 불러옵니다.
    try:
        api_key = st.secrets["KOBIS_KEY"]
    except Exception:
        return False, (
            "KOBIS_KEY를 찾을 수 없습니다. "
            "스트림릿 클라우드의 [Settings] → [Secrets]에 "
            'KOBIS_KEY = "발급받은키" 형식으로 등록했는지 확인해 주세요.'
        )

    params = {
        "key": api_key,
        "targetDt": target_dt,
    }

    # 네트워크 요청 자체가 실패하는 경우(타임아웃, 서버 다운 등)를 대비합니다.
    try:
        response = requests.get(API_URL, params=params, timeout=10)
    except requests.exceptions.RequestException:
        return False, (
            "KOBIS 서버에 접속하지 못했습니다. "
            "인터넷 연결 상태나 KOBIS 서버 상태를 확인해 주세요."
        )

    # 상태 코드가 200이 아닌 경우 (드물지만 서버 오류 등)
    if response.status_code != 200:
        return False, (
            f"KOBIS 서버가 오류를 반환했습니다 (상태코드 {response.status_code}). "
            "잠시 후 다시 시도해 주세요."
        )

    try:
        data = response.json()
    except ValueError:
        return False, "KOBIS 서버 응답을 해석할 수 없습니다. 잠시 후 다시 시도해 주세요."

    # 문서에 나온 대로, 인증키가 틀려도 상태코드는 200이고
    # 대신 faultInfo 상자가 옵니다. 이 경우를 반드시 확인해야 합니다.
    if "faultInfo" in data:
        message = data["faultInfo"].get("message", "알 수 없는 오류")
        return False, (
            f"KOBIS API가 오류를 반환했습니다: {message}. "
            "인증키(KOBIS_KEY)가 올바른지, 발급 승인이 끝났는지 확인해 주세요."
        )

    # 정상적인 경우 boxOfficeResult 안에 dailyBoxOfficeList가 들어 있습니다.
    box_office_result = data.get("boxOfficeResult")
    if not box_office_result:
        return False, "KOBIS 응답에 boxOfficeResult가 없습니다. API 문서가 바뀌었는지 확인해 주세요."

    movie_list = box_office_result.get("dailyBoxOfficeList")
    if not movie_list:
        return False, (
            "해당 날짜의 영화 목록이 비어 있습니다. "
            "조회 날짜가 너무 이르거나(집계 전), 공휴일 등으로 자료가 없을 수 있습니다."
        )

    return True, movie_list


# ------------------------------------------------------------
# 4. 문자열로 오는 숫자들을 실제 숫자(정수)로 바꾸기
# ------------------------------------------------------------
def to_dataframe(movie_list: list) -> pd.DataFrame:
    df = pd.DataFrame(movie_list)

    # API 문서에 명시된 대로 숫자 값들이 전부 문자열로 옵니다.
    # 정렬과 그래프에 쓰려면 숫자(정수) 타입으로 바꿔야 합니다.
    numeric_cols = ["rank", "audiCnt", "audiAcc", "scrnCnt", "showCnt"]
    for col in numeric_cols:
        if col in df.columns:
            df[col] = pd.to_numeric(df[col], errors="coerce")

    return df


# ------------------------------------------------------------
# 5. 화면 그리기
# ------------------------------------------------------------
st.title("🎬 어제의 박스오피스")

target_dt = get_yesterday_kst()
# yyyymmdd를 사람이 읽기 편한 형태(yyyy-mm-dd)로 바꿔서 보여줍니다.
pretty_date = f"{target_dt[:4]}-{target_dt[4:6]}-{target_dt[6:]}"
st.caption(f"조회 기준일(한국시간, 어제): {pretty_date}")

success, result = fetch_box_office(target_dt)

if not success:
    # 실패 사유를 화면에 안내 상자로 보여주고, 여기서 앱 실행을 멈춥니다.
    st.error(result)
    st.stop()

df = to_dataframe(result)

# rank 기준으로 정렬(문자열이 아니라 숫자 기준이라 1, 2, 3 ... 순서가 정확합니다)
df = df.sort_values("rank").reset_index(drop=True)

# ------------------------------------------------------------
# 5-1. 1위 영화 지표 카드 세 장
# ------------------------------------------------------------
top_movie = df.iloc[0]

st.subheader(f"오늘의 1위: {top_movie['movieNm']}")

col1, col2, col3 = st.columns(3)
col1.metric("어제 관객수", f"{int(top_movie['audiCnt']):,} 명")
col2.metric("누적 관객수", f"{int(top_movie['audiAcc']):,} 명")
col3.metric("스크린수", f"{int(top_movie['scrnCnt']):,} 개")

st.divider()

# ------------------------------------------------------------
# 5-2. 관객수 상위 5편 막대그래프
# ------------------------------------------------------------
st.subheader("관객수 상위 5편")

top5 = df.sort_values("audiCnt", ascending=False).head(5)

# 막대그래프용 데이터: 영화명을 인덱스로 두면 x축 라벨로 자동 사용됩니다.
chart_data = top5.set_index("movieNm")[["audiCnt"]]
chart_data.columns = ["어제 관객수"]

st.bar_chart(chart_data)

st.divider()

# ------------------------------------------------------------
# 5-3. 전체 순위 표
# ------------------------------------------------------------
st.subheader("전체 박스오피스 순위")

table_df = df[
    ["rank", "movieNm", "openDt", "audiCnt", "audiAcc", "scrnCnt"]
].rename(
    columns={
        "rank": "순위",
        "movieNm": "영화명",
        "openDt": "개봉일",
        "audiCnt": "관객수",
        "audiAcc": "누적관객",
        "scrnCnt": "스크린수",
    }
)

st.dataframe(
    table_df.style.format(
        {
            "관객수": "{:,}",
            "누적관객": "{:,}",
            "스크린수": "{:,}",
        }
    ),
    use_container_width=True,
    hide_index=True,
)

st.caption("데이터 출처: 영화진흥위원회(KOBIS) 일별 박스오피스 API")
