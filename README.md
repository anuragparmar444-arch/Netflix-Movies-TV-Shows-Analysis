
import pandas as pd
import numpy as np


def load_data(file_path="netflix_titles.csv"):
    """Load and prepare the Netflix dataset."""
    df = pd.read_csv(file_path)

    df.columns = df.columns.str.strip().str.lower()

    required = [
        "show_id", "type", "title", "country",
        "date_added", "release_year", "rating",
        "duration", "listed_in"
    ]

    missing = [col for col in required if col not in df.columns]
    if missing:
        raise ValueError(f"Missing required columns: {missing}")

    for col in df.select_dtypes(include="object").columns:
        df[col] = df[col].replace(r"^\s*$", np.nan, regex=True)

    df = df.drop_duplicates(subset=["show_id"])
    df["release_year"] = pd.to_numeric(
        df["release_year"], errors="coerce"
    )
    df["date_added"] = pd.to_datetime(
        df["date_added"], errors="coerce"
    )

    return df


def get_kpis(df):
    """Calculate key performance indicators."""
    total = len(df)
    movies = int((df["type"] == "Movie").sum())
    shows = int((df["type"] == "TV Show").sum())

    countries = (
        df["country"].dropna()
        .str.split(",")
        .explode()
        .str.strip()
        .nunique()
    )

    return {
        "total_titles": total,
        "movies": movies,
        "tv_shows": shows,
        "countries": int(countries),
        "missing_values": int(df.isna().sum().sum()),
        "movie_percentage": round(movies / total * 100, 1)
        if total else 0
    }


def get_top_countries(df, limit=10):
    countries = (
        df["country"].dropna()
        .str.split(",")
        .explode()
        .str.strip()
    )
    return countries.value_counts().head(limit)


def get_top_genres(df, limit=10):
    genres = (
        df["listed_in"].dropna()
        .str.split(",")
        .explode()
        .str.strip()
    )
    return genres.value_counts().head(limit)


def get_yearly_trend(df):
    return (
        df.dropna(subset=["date_added"])
        .assign(year=lambda x: x["date_added"].dt.year)
        .groupby("year")
        .size()
        .sort_index()
    )

4.interactive dashboards 

import streamlit as st
import pandas as pd
import plotly.express as px

from analysis import (
    load_data,
    get_kpis,
    get_top_countries,
    get_top_genres,
    get_yearly_trend,
)

st.set_page_config(
    page_title="Netflix Analytics Dashboard",
    page_icon="🎬",
    layout="wide"
)

# Dashboard theme
st.markdown("""
<style>
.stApp {
    background-color: #101010;
    color: white;
}
h1, h2, h3 {
    color: #E50914 !important;
}
[data-testid="stMetric"] {
    background: #202020;
    padding: 18px;
    border-radius: 12px;
    border: 1px solid #383838;
}
[data-testid="stMetricValue"] {
    color: white;
}
</style>
""", unsafe_allow_html=True)


@st.cache_data
def load_dataset():
    return load_data("netflix_titles.csv")


st.title("🎬 NETFLIX MOVIES & TV SHOWS")
st.subheader("Interactive Data Analytics Dashboard")
st.caption("Explore content types, release trends, countries and genres.")

# Load dataset
try:
    df = load_dataset()
except FileNotFoundError:
    st.error(
        "Dataset not found. Download netflix_titles.csv "
        "and place it in the same folder as app.py."
    )
    st.markdown(
        "[Download dataset information from Kaggle]"
        "(https://www.kaggle.com/datasets/shivamb/netflix-shows)"
    )
    st.stop()
except Exception as error:
    st.error(f"Unable to load dataset: {error}")
    st.stop()


# Sidebar filters
st.sidebar.title("🔎 Dashboard Filters")

types = ["All"] + sorted(df["type"].dropna().unique().tolist())
selected_type = st.sidebar.selectbox("Content Type", types)

years = df["release_year"].dropna()
if years.empty:
    min_year, max_year = 1900, 2026
else:
    min_year = int(years.min())
    max_year = int(years.max())

year_range = st.sidebar.slider(
    "Release Year Range",
    min_value=min_year,
    max_value=max_year,
    value=(min_year, max_year)
)

rating_values = sorted(df["rating"].dropna().unique().tolist())
selected_ratings = st.sidebar.multiselect(
    "Content Rating",
    options=rating_values,
    default=[]
)

search_text = st.sidebar.text_input("Search by Title")

# Apply filters
filtered = df.copy()

if selected_type != "All":
    filtered = filtered[filtered["type"] == selected_type]

filtered = filtered[
    filtered["release_year"].between(
        year_range[0], year_range[1]
    )
]

if selected_ratings:
    filtered = filtered[filtered["rating"].isin(selected_ratings)]

if search_text.strip():
    filtered = filtered[
        filtered["title"].str.contains(
            search_text.strip(), case=False, na=False, regex=False
        )
    ]

# KPI cards
total = len(filtered)
movies = int((filtered["type"] == "Movie").sum())
shows = int((filtered["type"] == "TV Show").sum())

country_count = (
    filtered["country"].dropna()
    .str.split(",")
    .explode()
    .str.strip()
    .nunique()
)

st.markdown("## 📊 Key Performance Indicators")

c1, c2, c3, c4 = st.columns(4)
c1.metric("Total Titles", f"{total:,}")
c2.metric("Movies", f"{movies:,}")
c3.metric("TV Shows", f"{shows:,}")
c4.metric("Countries", f"{country_count:,}")

st.divider()

if filtered.empty:
    st.warning("No titles match the selected filters.")
    st.stop()

# Charts
left, right = st.columns(2)

with left:
    st.markdown("### 🎥 Movies vs TV Shows")
    type_counts = filtered["type"].value_counts().rename_axis(
        "Content Type"
    ).reset_index(name="Titles")

    fig_type = px.pie(
        type_counts,
        names="Content Type",
        values="Titles",
        hole=0.55,
        color_discrete_sequence=["#E50914", "#777777"]
    )
    fig_type.update_layout(
        template="plotly_dark",
        paper_bgcolor="#101010",
        plot_bgcolor="#101010"
    )
    st.plotly_chart(fig_type, use_container_width=True)

with right:
    st.markdown("### 📅 Titles by Release Year")
    yearly = (
        filtered.dropna(subset=["release_year"])
        .groupby("release_year")
        .size()
        .reset_index(name="Titles")
    )
    yearly["release_year"] = yearly["release_year"].astype(int)

    fig_year = px.line(
        yearly,
        x="release_year",
        y="Titles",
        markers=True,
        labels={
            "release_year": "Release Year",
            "Titles": "Number of Titles"
        }
    )
    fig_year.update_traces(line_color="#E50914")
    fig_year.update_layout(
        template="plotly_dark",
        paper_bgcolor="#101010",
        plot_bgcolor="#101010"
    )
    st.plotly_chart(fig_year, use_container_width=True)

left, right = st.columns(2)

with left:
    st.markdown("### 🌍 Top 10 Countries")
    countries = get_top_countries(filtered, limit=10)

    country_df = countries.rename_axis(
        "Country"
    ).reset_index(name="Titles")

    fig_country = px.bar(
        country_df.sort_values("Titles"),
        x="Titles",
        y="Country",
        orientation="h",
        color="Titles",
        color_continuous_scale=["#5C0000", "#E50914"]
    )
    fig_country.update_layout(
        template="plotly_dark",
        paper_bgcolor="#101010",
        plot_bgcolor="#101010",
        coloraxis_showscale=False
    )
    st.plotly_chart(fig_country, use_container_width=True)

with right:
    st.markdown("### 🎭 Top 10 Genres")
    genres = get_top_genres(filtered, limit=10)

    genre_df = genres.rename_axis(
        "Genre"
    ).reset_index(name="Titles")

    fig_genre = px.bar(
        genre_df.sort_values("Titles"),
        x="Titles",
        y="Genre",
        orientation="h",
        color="Titles",
        color_continuous_scale=["#5C0000", "#E50914"]
    )
    fig_genre.update_layout(
        template="plotly_dark",
        paper_bgcolor="#101010",
        plot_bgcolor="#101010",
        coloraxis_showscale=False
    )
    st.plotly_chart(fig_genre, use_container_width=True)

# Ratings
st.markdown("### 🔞 Content Rating Distribution")

rating_df = (
    filtered["rating"].fillna("Unknown")
    .value_counts()
    .rename_axis("Rating")
    .reset_index(name="Titles")
)

fig_rating = px.bar(
    rating_df,
    x="Rating",
    y="Titles",
    color="Titles",
    color_continuous_scale=["#5C0000", "#E50914"]
)
fig_rating.update_layout(
    template="plotly_dark",
    paper_bgcolor="#101010",
    plot_bgcolor="#101010",
    coloraxis_showscale=False
)
st.plotly_chart(fig_rating, use_container_width=True)

# Data explorer
st.markdown("## 📚 Explore Netflix Titles")

display_columns = [
    col for col in [
        "show_id", "title", "type", "director", "country",
        "release_year", "rating", "duration", "listed_in",
        "description"
    ]
    if col in filtered.columns
]

st.dataframe(
    filtered[display_columns],
    use_container_width=True,
    hide_index=True
)

# Download filtered data
csv_data = filtered.to_csv(index=False).encode("utf-8-sig")

st.download_button(
    label="⬇️ Download Filtered Data as CSV",
    data=csv_data,
    file_name="netflix_filtered_analysis.csv",
    mime="text/csv"
)

st.divider()
st.caption(
    "Netflix Movies & TV Shows Analysis | "
    "Python • Pandas • Plotly • Streamlit"
)