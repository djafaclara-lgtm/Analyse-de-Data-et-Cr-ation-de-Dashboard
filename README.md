# Analyse-de-Data-et-Cr-ation-de-Dashboard
import numpy as np
import pandas as pd
import scipy.stats as stats


def generate_analysis(
    df: pd.DataFrame,
    detect_anomalies: bool = True,
    compute_corr: bool = True,
    detect_trends: bool = True,
    contamination: float = 0.05,
) -> dict:
    """Génère une analyse complète du DataFrame : stats, corrélations, anomalies, KPIs."""

    results: dict = {}

    num_cols  = df.select_dtypes(include="number").columns.tolist()
    cat_cols  = df.select_dtypes(include=["object", "category"]).columns.tolist()
    date_cols = df.select_dtypes(include="datetime").columns.tolist()

    results["num_cols"]  = num_cols
    results["cat_cols"]  = cat_cols
    results["date_cols"] = date_cols

    # ── Statistiques descriptives ─────────────────────────────────────────────
    if num_cols:
        desc = df[num_cols].describe().T
        desc["skewness"]  = df[num_cols].skew()
        desc["kurtosis"]  = df[num_cols].kurt()
        desc["nulls"]     = df[num_cols].isnull().sum()
        desc["nulls_pct"] = (df[num_cols].isnull().sum() / len(df) * 100).round(2)
        results["descriptive"] = desc

    # ── KPIs par colonne numérique ────────────────────────────────────────────
    kpis = {}
    for col in num_cols:
        s = df[col].dropna()
        kpis[col] = {
            "moyenne":      round(float(s.mean()), 4),
            "médiane":      round(float(s.median()), 4),
            "écart-type":   round(float(s.std()), 4),
            "min":          round(float(s.min()), 4),
            "max":          round(float(s.max()), 4),
            "skewness":     round(float(stats.skew(s)), 4),
            "kurtosis":     round(float(stats.kurtosis(s)), 4),
            "q1":           round(float(s.quantile(0.25)), 4),
            "q3":           round(float(s.quantile(0.75)), 4),
            "iqr":          round(float(s.quantile(0.75) - s.quantile(0.25)), 4),
        }
    results["kpis"] = kpis

    # ── Fréquences catégorielles ──────────────────────────────────────────────
    cat_freq = {}
    for col in cat_cols:
        freq = df[col].value_counts().head(15)
        cat_freq[col] = freq.reset_index().rename(
            columns={"index": col, col: "count"}
        )
    results["cat_freq"] = cat_freq

    # ── Matrice de corrélations ───────────────────────────────────────────────
    if compute_corr and len(num_cols) >= 2:
        corr = df[num_cols].corr(method="pearson")
        results["corr_matrix"] = corr

        # Top paires corrélées (hors diagonale)
        mask = np.triu(np.ones(corr.shape, dtype=bool), k=1)
        corr_vals = corr.where(mask).stack().reset_index()
        corr_vals.columns = ["col_a", "col_b", "correlation"]
        corr_vals["abs_corr"] = corr_vals["correlation"].abs()
        results["top_correlations"] = (
            corr_vals.nlargest(10, "abs_corr").reset_index(drop=True)
        )

    # ── Détection d'anomalies (Isolation Forest) ──────────────────────────────
    results["anomaly_count"] = 0
    results["anomaly_rows"]  = pd.DataFrame()

    if detect_anomalies and num_cols and len(df) >= 20:
        try:
            from sklearn.ensemble import IsolationForest
            from sklearn.preprocessing import RobustScaler

            X = df[num_cols].fillna(df[num_cols].median())
            X_scaled = RobustScaler().fit_transform(X)

            iso = IsolationForest(
                contamination=contamination,
                random_state=42,
                n_estimators=100
            )
            preds = iso.fit_predict(X_scaled)
            scores = iso.score_samples(X_scaled)

            df_tmp = df.copy()
            df_tmp["_anomaly_flag"]  = preds
            df_tmp["_anomaly_score"] = scores.round(4)

            anomaly_df = df_tmp[df_tmp["_anomaly_flag"] == -1].drop(
                "_anomaly_flag", axis=1
            ).sort_values("_anomaly_score")

            results["anomaly_count"] = len(anomaly_df)
            results["anomaly_rows"]  = anomaly_df
        except ImportError:
            results["anomaly_count"] = -1  # sklearn non dispo

    # ── Tendances temporelles ─────────────────────────────────────────────────
    if detect_trends and date_cols and num_cols:
        date_col = date_cols[0]
        df_sorted = df[[date_col] + num_cols].sort_values(date_col).dropna(subset=[date_col])

        # Resampling mensuel si assez de données
        try:
            df_ts = df_sorted.set_index(date_col)
            df_monthly = df_ts[num_cols].resample("ME").mean().reset_index()
            results["time_series"] = {
                "date_col":  date_col,
                "value_cols": num_cols,
                "df_raw":    df_sorted,
                "df_monthly": df_monthly,
            }
        except Exception:
            results["time_series"] = {
                "date_col":   date_col,
                "value_cols": num_cols,
                "df_raw":     df_sorted,
                "df_monthly": df_sorted,
            }

    return results
    import streamlit as st
from data_loader import smart_load, clean_dataframe
from analyzer import generate_analysis
from dashboard import render_dashboard
from exporter import export_to_excel

st.set_page_config(
    page_title="DataLens",
    page_icon="🔍",
    layout="wide",
    initial_sidebar_state="expanded"
)

# CSS personnalisé
st.markdown("""
<style>
    .main-title { font-size: 2.2rem; font-weight: 700; color: #4F46E5; }
    .subtitle   { font-size: 1rem; color: #6B7280; margin-top: -10px; }
    .kpi-card   { background: #F9FAFB; border-radius: 10px; padding: 16px;
                  border-left: 4px solid #4F46E5; }
    .section-title { font-size: 1.1rem; font-weight: 600; color: #374151;
                     margin: 24px 0 8px; }
    div[data-testid="metric-container"] {
        background: #F9FAFB; border-radius: 10px; padding: 12px;
        border: 1px solid #E5E7EB;
    }
</style>
""", unsafe_allow_html=True)

# ── Sidebar ──────────────────────────────────────────────────────────────────
with st.sidebar:
    st.image("https://img.icons8.com/fluency/96/combo-chart.png", width=60)
    st.markdown("## 🔍 DataLens")
    st.markdown("Analyse automatique de données")
    st.divider()

    st.markdown("### ⚙️ Options d'analyse")
    detect_anomalies  = st.toggle("Détection d'anomalies", value=True)
    compute_corr      = st.toggle("Matrice de corrélations", value=True)
    detect_trends     = st.toggle("Tendances temporelles", value=True)
    contamination_pct = st.slider("Seuil anomalies (%)", 1, 20, 5) / 100

    st.divider()
    st.markdown("### 📤 Export")
    max_chart_sheets = st.slider("Feuilles graphiques max", 1, 10, 5)

    st.divider()
    st.caption("DataLens v1.0 · Propulsé par Streamlit")

# ── Header ───────────────────────────────────────────────────────────────────
st.markdown('<p class="main-title">🔍 DataLens</p>', unsafe_allow_html=True)
st.markdown('<p class="subtitle">Import · Analyse · Dashboard · Export Excel</p>',
            unsafe_allow_html=True)
st.divider()

# ── Upload ───────────────────────────────────────────────────────────────────
uploaded = st.file_uploader(
    "📂 Dépose ton fichier ici",
    type=["csv", "tsv", "xlsx", "xls", "json", "parquet"],
    help="Formats supportés : CSV, TSV, Excel (.xlsx/.xls), JSON, Parquet"
)

if uploaded is None:
    st.info("👆 Importe un fichier pour démarrer l'analyse automatique.", icon="💡")
    with st.expander("ℹ️ Fonctionnalités disponibles"):
        col1, col2, col3 = st.columns(3)
        with col1:
            st.markdown("**📊 Analyse**")
            st.markdown("- Statistiques descriptives\n- Corrélations\n- Anomalies\n- Tendances\n- KPIs auto")
        with col2:
            st.markdown("**📈 Visualisations**")
            st.markdown("- Histogrammes\n- Heatmap\n- Courbes\n- Boxplots\n- KPI cards")
        with col3:
            st.markdown("**📤 Export**")
            st.markdown("- Excel multi-feuilles\n- Graphiques natifs\n- Synthèse exécutive\n- Données nettoyées\n- Rapport de corrélations")
    st.stop()

# ── Chargement ───────────────────────────────────────────────────────────────
with st.spinner("⏳ Chargement et détection des types…"):
    df = smart_load(uploaded)

if df is None:
    st.stop()

with st.spinner("🧹 Nettoyage des données…"):
    df, clean_report = clean_dataframe(df)

# Métriques rapides
c1, c2, c3, c4 = st.columns(4)
c1.metric("📋 Lignes",    f"{len(df):,}")
c2.metric("📐 Colonnes",  f"{len(df.columns)}")
c3.metric("🗑️ Doublons supprimés", clean_report.get("doublons_supprimés", 0))
nulls = sum(v for v in clean_report.get("valeurs_nulles", {}).values())
c4.metric("🔧 Valeurs nulles traitées", nulls)

with st.expander("📋 Rapport de nettoyage détaillé"):
    st.json(clean_report)
    st.dataframe(df.head(10), use_container_width=True)

st.divider()

# ── Analyse ───────────────────────────────────────────────────────────────────
with st.spinner("🔬 Analyse statistique en cours…"):
    analysis = generate_analysis(
        df,
        detect_anomalies=detect_anomalies,
        compute_corr=compute_corr,
        detect_trends=detect_trends,
        contamination=contamination_pct
    )

# ── Dashboard ─────────────────────────────────────────────────────────────────
render_dashboard(df, analysis)

st.divider()

# ── Export ────────────────────────────────────────────────────────────────────
st.markdown('<p class="section-title">📤 Exporter le rapport</p>',
            unsafe_allow_html=True)

with st.spinner("📊 Génération du fichier Excel…"):
    excel_bytes = export_to_excel(df, analysis, max_charts=max_chart_sheets)

st.download_button(
    label="⬇️ Télécharger le rapport Excel complet",
    data=excel_bytes,
    file_name=f"datalens_rapport_{uploaded.name.split('.')[0]}.xlsx",
    mime="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
    use_container_width=True,
    type="primary"
)
import pandas as pd
import streamlit as st
from pathlib import Path

SUPPORTED_FORMATS = {
    ".csv":     lambda f, **kw: pd.read_csv(f, **kw),
    ".tsv":     lambda f, **kw: pd.read_csv(f, sep="\t", **kw),
    ".xlsx":    lambda f, **kw: pd.read_excel(f, **kw),
    ".xls":     lambda f, **kw: pd.read_excel(f, **kw),
    ".json":    lambda f, **kw: pd.read_json(f, **kw),
    ".parquet": lambda f, **kw: pd.read_parquet(f, **kw),
}


def smart_load(uploaded_file) -> pd.DataFrame | None:
    """Charge un fichier uploadé et détecte automatiquement les types de colonnes."""
    ext = Path(uploaded_file.name).suffix.lower()

    if ext not in SUPPORTED_FORMATS:
        st.error(f"❌ Format « {ext} » non supporté. "
                 f"Formats acceptés : {', '.join(SUPPORTED_FORMATS)}")
        return None

    try:
        df = SUPPORTED_FORMATS[ext](uploaded_file)
    except Exception as e:
        st.error(f"❌ Erreur lors du chargement : {e}")
        return None

    if df.empty:
        st.warning("⚠️ Le fichier est vide ou ne contient pas de données lisibles.")
        return None

    # Nettoyage préliminaire des noms de colonnes
    df.columns = df.columns.astype(str).str.strip()

    # Détection intelligente des types colonne par colonne
    for col in df.columns:
        if df[col].dtype != object:
            continue

        # 1) Tentative datetime
        try:
            converted = pd.to_datetime(df[col], infer_datetime_format=True, errors="raise")
            df[col] = converted
            continue
        except (ValueError, TypeError, OverflowError):
            pass

        # 2) Tentative numérique (gère "1 000,50" → 1000.50)
        cleaned = (
            df[col].astype(str)
            .str.strip()
            .str.replace(r"\s", "", regex=True)   # espaces (séparateurs milliers)
            .str.replace(",", ".", regex=False)    # virgule décimale FR
        )
        try:
            numeric = pd.to_numeric(cleaned, errors="raise")
            df[col] = numeric
        except (ValueError, TypeError):
            pass  # reste en object (catégorielle / texte)

    return df


def clean_dataframe(df: pd.DataFrame) -> tuple[pd.DataFrame, dict]:
    """Nettoie le DataFrame et retourne un rapport de nettoyage."""
    report: dict = {}
    df = df.copy()

    # 1. Suppression des colonnes entièrement vides
    empty_cols = df.columns[df.isnull().all()].tolist()
    df.drop(columns=empty_cols, inplace=True)
    report["colonnes_vides_supprimées"] = empty_cols

    # 2. Suppression des doublons
    before = len(df)
    df.drop_duplicates(inplace=True)
    report["doublons_supprimés"] = before - len(df)

    # 3. Rapport valeurs nulles (avant imputation)
    nulls = df.isnull().sum()
    report["valeurs_nulles"] = nulls[nulls > 0].to_dict()

    # 4. Imputation
    for col in df.select_dtypes(include="number").columns:
        if df[col].isnull().any():
            df[col].fillna(df[col].median(), inplace=True)

    for col in df.select_dtypes(include=["object", "category"]).columns:
        if df[col].isnull().any():
            mode = df[col].mode()
            df[col].fillna(mode[0] if not mode.empty else "N/A", inplace=True)

    for col in df.select_dtypes(include="datetime").columns:
        if df[col].isnull().any():
            df[col].fillna(method="ffill", inplace=True)

    # 5. Normalisation des noms de colonnes (snake_case)
    df.columns = [
        c.strip().lower()
         .replace(" ", "_")
         .replace("-", "_")
         .replace("(", "")
         .replace(")", "")
        for c in df.columns
    ]

    report["nb_lignes_final"]    = len(df)
    report["nb_colonnes_final"]  = len(df.columns)
    report["types_détectés"]     = df.dtypes.astype(str).to_dict()

    return df, report
import io
from datetime import datetime

import numpy as np
import openpyxl
import pandas as pd
from openpyxl.chart import BarChart, LineChart, Reference
from openpyxl.chart.series import DataPoint
from openpyxl.styles import (Alignment, Border, Font, GradientFill,
                              PatternFill, Side)
from openpyxl.utils import get_column_letter
from openpyxl.utils.dataframe import dataframe_to_rows

# ── Constantes de style ───────────────────────────────────────────────────────
INDIGO     = "4F46E5"
INDIGO_LT  = "EEF2FF"
GRAY       = "F3F4F6"
GRAY_DARK  = "6B7280"
WHITE      = "FFFFFF"
RED        = "EF4444"
GREEN      = "22C55E"
ORANGE     = "F97316"

FONT_MAIN  = "Calibri"


def _font(bold=False, size=11, color=None):
    return Font(name=FONT_MAIN, bold=bold, size=size,
                color=color or "000000")

def _fill(hex_color):
    return PatternFill("solid", fgColor=hex_color)

def _align(h="center", v="center", wrap=False):
    return Alignment(horizontal=h, vertical=v, wrap_text=wrap)

def _border(sides="all", style="thin"):
    side = Side(style=style)
    if sides == "all":
        return Border(left=side, right=side, top=side, bottom=side)
    if sides == "bottom":
        return Border(bottom=side)
    return Border()

def _style_header(ws, row: int, n_cols: int, bg=INDIGO, fg=WHITE):
    for c in range(1, n_cols + 1):
        cell = ws.cell(row=row, column=c)
        cell.font      = _font(bold=True, color=fg)
        cell.fill      = _fill(bg)
        cell.alignment = _align()
        cell.border    = _border()

def _style_data_rows(ws, start_row: int, end_row: int, n_cols: int):
    for r in range(start_row, end_row + 1):
        bg = GRAY if r % 2 == 0 else WHITE
        for c in range(1, n_cols + 1):
            cell = ws.cell(row=r, column=c)
            cell.fill      = _fill(bg)
            cell.border    = _border(sides="bottom", style="hair")
            cell.alignment = _align(h="left", v="center")

def _autofit(ws, min_w=8, max_w=40):
    for col_cells in ws.columns:
        length = max(
            len(str(cell.value or "")) for cell in col_cells
        )
        ws.column_dimensions[get_column_letter(col_cells[0].column)].width = (
            min(max(length + 2, min_w), max_w)
        )


# ── Feuille Synthèse ──────────────────────────────────────────────────────────
def _add_summary_sheet(wb, df, analysis):
    ws = wb.active
    ws.title = "Synthèse"
    ws.sheet_view.showGridLines = False
    ws.column_dimensions["A"].width = 30
    ws.column_dimensions["B"].width = 20

    # Titre principal
    ws.merge_cells("A1:D1")
    ws["A1"] = "RAPPORT D'ANALYSE — DataLens"
    ws["A1"].font      = _font(bold=True, size=16, color=INDIGO)
    ws["A1"].alignment = _align(h="left")
    ws["A2"] = f"Généré le {datetime.now().strftime('%d/%m/%Y à %H:%M')}"
    ws["A2"].font      = _font(size=10, color=GRAY_DARK)
    ws.row_dimensions[1].height = 30

    ws.append([])  # ligne vide

    # Métriques clés
    metrics = [
        ("📋 Nombre de lignes",       f"{len(df):,}"),
        ("📐 Nombre de colonnes",      str(len(df.columns))),
        ("🔢 Colonnes numériques",     str(len(analysis.get("num_cols", [])))),
        ("🗂️ Colonnes catégorielles",  str(len(analysis.get("cat_cols", [])))),
        ("📅 Colonnes date",           str(len(analysis.get("date_cols", [])))),
        ("⚠️ Anomalies détectées",     str(analysis.get("anomaly_count", "N/A"))),
    ]
    ws.append(["Indicateur", "Valeur"])
    _style_header(ws, ws.max_row, 2)

    for label, val in metrics:
        ws.append([label, val])
    _style_data_rows(ws, ws.max_row - len(metrics) + 1, ws.max_row, 2)

    ws.append([])

    # Top corrélations
    top_corr = analysis.get("top_correlations")
    if top_corr is not None and not top_corr.empty:
        ws.append(["Top corrélations"])
        ws[f"A{ws.max_row}"].font = _font(bold=True, size=12, color=INDIGO)
        ws.append(["Colonne A", "Colonne B", "Corrélation (Pearson)"])
        _style_header(ws, ws.max_row, 3, bg="6366F1")
        for _, row in top_corr.head(5).iterrows():
            ws.append([row["col_a"], row["col_b"], round(row["correlation"], 4)])
        _style_data_rows(ws, ws.max_row - 4, ws.max_row, 3)


# ── Feuille Données nettoyées ─────────────────────────────────────────────────
def _add_data_sheet(wb, df):
    ws = wb.create_sheet("Données")
    ws.sheet_view.showGridLines = False

    for r in dataframe_to_rows(df, index=False, header=True):
        ws.append(r)

    _style_header(ws, 1, len(df.columns))
    _style_data_rows(ws, 2, len(df) + 1, len(df.columns))
    _autofit(ws)

    # Freeze pane (header fixe)
    ws.freeze_panes = "A2"


# ── Feuille KPIs ──────────────────────────────────────────────────────────────
def _add_kpi_sheet(wb, kpis: dict):
    if not kpis:
        return
    ws = wb.create_sheet("KPIs")
    ws.sheet_view.showGridLines = False

    headers = ["Colonne", "Moyenne", "Médiane", "Écart-type",
               "Min", "Max", "Q1", "Q3", "IQR", "Skewness", "Kurtosis"]
    ws.append(headers)
    _style_header(ws, 1, len(headers), bg="059669")

    for col_name, vals in kpis.items():
        ws.append([
            col_name,
            vals.get("moyenne"),
            vals.get("médiane"),
            vals.get("écart-type"),
            vals.get("min"),
            vals.get("max"),
            vals.get("q1"),
            vals.get("q3"),
            vals.get("iqr"),
            vals.get("skewness"),
            vals.get("kurtosis"),
        ])

    _style_data_rows(ws, 2, ws.max_row, len(headers))

    # Format numérique
    for row in ws.iter_rows(min_row=2, min_col=2):
        for cell in row:
            if isinstance(cell.value, (int, float)):
                cell.number_format = "#,##0.0000"

    _autofit(ws)


# ── Feuille Corrélations ───────────────────────────────────────────────────────
def _add_corr_sheet(wb, corr_matrix: pd.DataFrame):
    ws = wb.create_sheet("Corrélations")
    ws.sheet_view.showGridLines = False

    cols = corr_matrix.columns.tolist()
    ws.append([""] + cols)
    _style_header(ws, 1, len(cols) + 1, bg="7C3AED")

    for idx_name, row_data in corr_matrix.iterrows():
        ws.append([idx_name] + [round(v, 4) for v in row_data.values])

    # Colorisation conditionnelle manuelle
    for r in range(2, len(cols) + 2):
        for c in range(2, len(cols) + 2):
            cell = ws.cell(row=r, column=c)
            val  = cell.value or 0
            if val is None:
                continue
            if val >= 0.7:
                cell.fill = _fill("BBDEFB")
                cell.font = _font(color="0D47A1")
            elif val >= 0.4:
                cell.fill = _fill("E3F2FD")
            elif val <= -0.7:
                cell.fill = _fill("FFCDD2")
                cell.font = _font(color="B71C1C")
            elif val <= -0.4:
                cell.fill = _fill("FFEBEE")
            cell.number_format = "0.00"
            cell.alignment = _align()

    _autofit(ws, max_w=16)


# ── Feuilles Graphiques ────────────────────────────────────────────────────────
def _add_chart_sheet(wb, df: pd.DataFrame, col: str, chart_idx: int):
    safe_name = f"Graph {chart_idx} - {col[:18]}"
    ws = wb.create_sheet(safe_name)
    ws.sheet_view.showGridLines = False

    # Données source
    ws["A1"] = "Index"
    ws["B1"] = col
    _style_header(ws, 1, 2)

    series = df[col].dropna().reset_index(drop=True)
    for i, val in enumerate(series, 2):
        ws.cell(row=i, column=1, value=i - 1)
        ws.cell(row=i, column=2, value=round(float(val), 4))

    n_rows = len(series) + 1

    # Graphique en courbe
    chart = LineChart()
    chart.title       = f"Évolution — {col}"
    chart.style       = 10
    chart.y_axis.title = col
    chart.x_axis.title = "Index"
    chart.height       = 14
    chart.width        = 26

    data = Reference(ws, min_col=2, min_row=1, max_row=n_rows)
    cats = Reference(ws, min_col=1, min_row=2, max_row=n_rows)
    chart.add_data(data, titles_from_data=True)
    chart.set_categories(cats)
    chart.series[0].graphicalProperties.line.solidFill = INDIGO
    chart.series[0].graphicalProperties.line.width     = 20000

    ws.add_chart(chart, "D2")


# ── Feuille Anomalies ──────────────────────────────────────────────────────────
def _add_anomaly_sheet(wb, anomaly_df: pd.DataFrame):
    if anomaly_df.empty:
        return
    ws = wb.create_sheet("Anomalies")
    ws.sheet_view.showGridLines = False

    ws["A1"] = f"⚠️ {len(anomaly_df)} anomalies détectées"
    ws["A1"].font = _font(bold=True, size=13, color=RED)
    ws.append([])

    cols = anomaly_df.columns.tolist()
    ws.append(cols)
    _style_header(ws, ws.max_row, len(cols), bg=RED)

    for _, row in anomaly_df.iterrows():
        ws.append(row.tolist())

    _style_data_rows(ws, ws.max_row - len(anomaly_df) + 1, ws.max_row, len(cols))
    _autofit(ws)


# ── Export principal ───────────────────────────────────────────────────────────
def export_to_excel(df: pd.DataFrame, analysis: dict, max_charts: int = 5) -> bytes:
    wb = openpyxl.Workbook()

    _add_summary_sheet(wb, df, analysis)
    _add_data_sheet(wb, df)

    if analysis.get("kpis"):
        _add_kpi_sheet(wb, analysis["kpis"])

    if "corr_matrix" in analysis:
        _add_corr_sheet(wb, analysis["corr_matrix"])

    num_cols = analysis.get("num_cols", [])
    for i, col in enumerate(num_cols[:max_charts], 1):
        _add_chart_sheet(wb, df, col, i)

    anomaly_df = analysis.get("anomaly_rows", pd.DataFrame())
    if not anomaly_df.empty:
        _add_anomaly_sheet(wb, anomaly_df)

    buf = io.BytesIO()
    wb.save(buf)
    return buf.getvalue()
# 🔍 DataLens — Analyse automatique de données

Application Streamlit pour importer, analyser et exporter des jeux de données en quelques clics.

## 🚀 Installation rapide

### 1. Prérequis
- Python 3.10+ installé ([python.org](https://www.python.org/downloads/))

### 2. Installer les dépendances

```bash
pip install -r requirements.txt
```

### 3. Lancer l'application

```bash
streamlit run app.py
```

L'app s'ouvre automatiquement sur **http://localhost:8501**

---

## 📁 Structure du projet

```
datalens/
├── app.py           → Interface principale Streamlit
├── data_loader.py   → Import & nettoyage des fichiers
├── analyzer.py      → Moteur d'analyse statistique
├── dashboard.py     → Visualisations Plotly
├── exporter.py      → Export Excel multi-feuilles
└── requirements.txt → Dépendances Python
```

## 📊 Formats supportés

| Format | Extension |
|--------|-----------|
| Excel  | .xlsx, .xls |
| CSV    | .csv, .tsv |
| JSON   | .json |
| Parquet | .parquet |

## ✨ Fonctionnalités

- **Import intelligent** : détection auto des types (dates, nombres, catégories)
- **Nettoyage auto** : doublons, valeurs nulles, normalisation
- **Statistiques** : descriptives, corrélations, skewness, kurtosis
- **Anomalies** : Isolation Forest avec score de confiance
- **Tendances** : séries temporelles avec resampling mensuel
- **Dashboard interactif** : 6 onglets de visualisations Plotly
- **Export Excel** : rapport multi-feuilles avec graphiques natifs

## 🛠️ Options disponibles (sidebar)

- Activer/désactiver chaque module d'analyse
- Ajuster le seuil de détection d'anomalies (1–20%)
- Contrôler le nombre de feuilles graphiques dans l'export

---

*DataLens v1.0 — Propulsé par Streamlit, Pandas & Plotly*
streamlit>=1.35.0
pandas>=2.2.0
numpy>=1.26.0
plotly>=5.22.0
scipy>=1.13.0
scikit-learn>=1.5.0
openpyxl>=3.1.0
pyarrow>=16.0.0
statsmodels>=0.14.0

