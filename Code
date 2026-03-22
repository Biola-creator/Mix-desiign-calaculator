import math
from dataclasses import dataclass
from typing import Dict, List

import pandas as pd
import streamlit as st

st.set_page_config(page_title="Mix Design MVP", layout="wide")

# ----------------------------
# Constants and starter data
# ----------------------------
KG_TO_LB = 2.20462262185
M3_TO_YD3 = 1.30795061931
L_TO_GAL = 0.26417205236

DEFAULT_PROPS = {
    "OPC": {"specific_gravity": 3.15, "cost_per_kg_usd": 0.14, "carbon_kgco2e_per_kg": 0.82},
    "Slag": {"specific_gravity": 2.90, "cost_per_kg_usd": 0.10, "carbon_kgco2e_per_kg": 0.07},
    "Metakaolin": {"specific_gravity": 2.20, "cost_per_kg_usd": 0.45, "carbon_kgco2e_per_kg": 0.45},
    "Sand": {"specific_gravity": 2.65, "cost_per_kg_usd": 0.015, "carbon_kgco2e_per_kg": 0.005},
    "Coarse Aggregate": {"specific_gravity": 2.65, "cost_per_kg_usd": 0.020, "carbon_kgco2e_per_kg": 0.005},
    "Water": {"specific_gravity": 1.00, "cost_per_kg_usd": 0.001, "carbon_kgco2e_per_kg": 0.0003},
    "SP": {"specific_gravity": 1.08, "cost_per_kg_usd": 3.50, "carbon_kgco2e_per_kg": 1.20},
}

# Approximate ACI-inspired starter values for non-air-entrained concrete with angular coarse aggregate.
# Kept editable in the code so you can tune them later.
CONCRETE_WATER_KG_M3 = {
    9.5: {"Low": 207, "Medium": 228, "High": 243},
    12.5: {"Low": 196, "Medium": 216, "High": 228},
    19.0: {"Low": 186, "Medium": 205, "High": 216},
    25.0: {"Low": 177, "Medium": 195, "High": 205},
}

# Approximate bulk volume of dry-rodded coarse aggregate per m3 of concrete, assuming FM ~2.7.
COARSE_BULK_VOL = {9.5: 0.50, 12.5: 0.59, 19.0: 0.66, 25.0: 0.71}

MORTAR_WATER_KG_M3 = {"Low": 210, "Medium": 240, "High": 270}


# ----------------------------
# Helper functions
# ----------------------------
def recommended_wb_concrete(target_mpa: float) -> float:
    """Simple starter curve from target strength to a recommended maximum w/b."""
    return max(0.35, min(0.60, 0.70 - 0.005 * target_mpa))


def recommended_wb_mortar(target_mpa: float) -> float:
    """Mortar strength can be lower, so allow a broader practical window."""
    return max(0.35, min(0.70, 0.75 - 0.0065 * target_mpa))


def fmt_qty(value_kg: float, unit_system: str, material: str = "solid") -> str:
    if unit_system == "SI":
        if material == "water":
            return f"{value_kg:,.1f} kg ({value_kg:,.1f} L)"
        return f"{value_kg:,.1f} kg"
    if material == "water":
        return f"{value_kg * KG_TO_LB:,.1f} lb ({value_kg * L_TO_GAL:,.1f} gal)"
    return f"{value_kg * KG_TO_LB:,.1f} lb"


def unit_rate_label(unit_system: str) -> str:
    return "kg/m³" if unit_system == "SI" else "lb/yd³"


def batch_volume_label(unit_system: str) -> str:
    return "m³" if unit_system == "SI" else "yd³"


def normalize_per_selected_unit(total_kg: float, batch_m3: float, unit_system: str) -> float:
    if batch_m3 <= 0:
        return 0.0
    if unit_system == "SI":
        return total_kg / batch_m3
    batch_yd3 = batch_m3 * M3_TO_YD3
    return total_kg * KG_TO_LB / batch_yd3


def totals_from_results(results: Dict[str, float], props_df: pd.DataFrame) -> Dict[str, float]:
    total_cost = 0.0
    total_carbon = 0.0
    for material, qty in results.items():
        if material in props_df.index:
            total_cost += qty * float(props_df.loc[material, "cost_per_kg_usd"])
            total_carbon += qty * float(props_df.loc[material, "carbon_kgco2e_per_kg"])
    return {"cost_usd": total_cost, "carbon_kgco2e": total_carbon}


def build_output_table(results: Dict[str, float], props_df: pd.DataFrame, batch_m3: float, unit_system: str) -> pd.DataFrame:
    rows = []
    for material, qty in results.items():
        per_selected = normalize_per_selected_unit(qty, batch_m3, unit_system)
        rows.append(
            {
                "Material": material,
                "Batch quantity": fmt_qty(qty, unit_system, "water" if material == "Water" else "solid"),
                f"Rate ({unit_rate_label(unit_system)})": round(per_selected, 2),
                "Unit cost (USD/kg)": float(props_df.loc[material, "cost_per_kg_usd"]),
                "Unit carbon (kg CO₂e/kg)": float(props_df.loc[material, "carbon_kgco2e_per_kg"]),
                "Batch cost (USD)": round(qty * float(props_df.loc[material, "cost_per_kg_usd"]), 2),
                "Batch carbon (kg CO₂e)": round(qty * float(props_df.loc[material, "carbon_kgco2e_per_kg"]), 2),
            }
        )
    return pd.DataFrame(rows)


# ----------------------------
# Mix design engines
# ----------------------------
def design_concrete(
    target_strength_mpa: float,
    wb: float,
    workability: str,
    nm_size_mm: float,
    batch_m3: float,
    binder_pct: Dict[str, float],
    props_df: pd.DataFrame,
    dry_rodded_unit_weight_kg_m3: float,
    air_content_pct: float,
    sp_dosage_pct: float,
) -> Dict[str, float]:
    water = CONCRETE_WATER_KG_M3[nm_size_mm][workability]
    binder_total = water / wb

    opc = binder_total * binder_pct["OPC"] / 100.0
    slag = binder_total * binder_pct["Slag"] / 100.0
    mk = binder_total * binder_pct["Metakaolin"] / 100.0

    bulk_vol = COARSE_BULK_VOL[nm_size_mm]
    if workability == "Low":
        bulk_vol += 0.01
    elif workability == "High":
        bulk_vol -= 0.01

    coarse_aggr = bulk_vol * dry_rodded_unit_weight_kg_m3

    water_vol = water / 1000.0
    air_vol = air_content_pct / 100.0
    binder_vol = (
        opc / (float(props_df.loc["OPC", "specific_gravity"]) * 1000.0)
        + slag / (float(props_df.loc["Slag", "specific_gravity"]) * 1000.0)
        + mk / (float(props_df.loc["Metakaolin", "specific_gravity"]) * 1000.0)
    )
    coarse_vol = coarse_aggr / (float(props_df.loc["Coarse Aggregate", "specific_gravity"]) * 1000.0)
    fine_vol = 1.0 - (water_vol + air_vol + binder_vol + coarse_vol)
    sand = max(0.0, fine_vol * float(props_df.loc["Sand", "specific_gravity"]) * 1000.0)

    results_per_m3 = {
        "OPC": opc,
        "Slag": slag,
        "Metakaolin": mk,
        "Water": water,
        "Sand": sand,
        "Coarse Aggregate": coarse_aggr,
    }

    if wb < 0.40:
        sp = binder_total * sp_dosage_pct / 100.0
        results_per_m3["SP"] = sp

    return {k: v * batch_m3 for k, v in results_per_m3.items() if v > 1e-9}



def design_mortar(
    target_strength_mpa: float,
    wb: float,
    workability: str,
    batch_m3: float,
    binder_pct: Dict[str, float],
    props_df: pd.DataFrame,
    sand_to_binder_vol_ratio: float,
    air_content_pct: float,
    sp_dosage_pct: float,
) -> Dict[str, float]:
    water_seed = MORTAR_WATER_KG_M3[workability]
    binder_total_seed = water_seed / wb

    opc_seed = binder_total_seed * binder_pct["OPC"] / 100.0
    slag_seed = binder_total_seed * binder_pct["Slag"] / 100.0
    mk_seed = binder_total_seed * binder_pct["Metakaolin"] / 100.0

    binder_vol = (
        opc_seed / (float(props_df.loc["OPC", "specific_gravity"]) * 1000.0)
        + slag_seed / (float(props_df.loc["Slag", "specific_gravity"]) * 1000.0)
        + mk_seed / (float(props_df.loc["Metakaolin", "specific_gravity"]) * 1000.0)
    )
    sand_vol = sand_to_binder_vol_ratio * binder_vol
    water_vol = water_seed / 1000.0
    air_vol = air_content_pct / 100.0

    seed_total_vol = binder_vol + sand_vol + water_vol + air_vol
    scale = batch_m3 / seed_total_vol if seed_total_vol > 0 else 0.0

    opc = opc_seed * scale
    slag = slag_seed * scale
    mk = mk_seed * scale
    water = water_seed * scale
    sand = sand_vol * float(props_df.loc["Sand", "specific_gravity"]) * 1000.0 * scale

    results = {
        "OPC": opc,
        "Slag": slag,
        "Metakaolin": mk,
        "Water": water,
        "Sand": sand,
    }

    if wb < 0.40:
        binder_total = opc + slag + mk
        results["SP"] = binder_total * sp_dosage_pct / 100.0

    return {k: v for k, v in results.items() if v > 1e-9}


# ----------------------------
# UI
# ----------------------------
st.title("Mix Design Calculator MVP")
st.caption("ACI-inspired starter app for mortar and concrete mix proportioning, with editable cost and carbon assumptions.")

st.markdown(
    """
This MVP estimates material quantities, total cost, embodied carbon, and warning flags.
Use it as a first-pass design tool and then calibrate it with your lab or field trial batches.
"""
)

with st.sidebar:
    st.header("Project setup")
    unit_system = st.selectbox("Unit system", ["SI", "US"], index=0)
    mix_type = st.selectbox("Mix type", ["Concrete", "Mortar"], index=0)
    batch_volume_input = st.number_input(
        f"Desired batch volume ({batch_volume_label(unit_system)})",
        min_value=0.01,
        value=1.0,
        step=0.1,
    )
    batch_m3 = batch_volume_input if unit_system == "SI" else batch_volume_input / M3_TO_YD3

    st.header("Editable material defaults")
    starter_df = pd.DataFrame(DEFAULT_PROPS).T
    starter_df.index.name = "Material"
    props_df = st.data_editor(
        starter_df,
        use_container_width=True,
        num_rows="fixed",
        key="props_editor",
    )

col1, col2 = st.columns([1.2, 1])

with col1:
    st.subheader("Design inputs")
    if mix_type == "Concrete":
        target_strength = st.slider("Target compressive strength (MPa)", 25, 60, 35)
    else:
        target_strength = st.slider("Target compressive strength (MPa)", 5, 60, 20)
        st.info("Mortar strength slider starts lower than concrete, per your requirement.")

    wb = st.slider("Water-to-binder ratio (w/b)", 0.35, 0.60, 0.45, 0.01)
    if wb < 0.40:
        st.warning("w/b below 0.40: superplasticizer should be added.")
    else:
        st.caption("Normal starter range is 0.40 to 0.60. Below 0.40 usually needs SP.")

    if mix_type == "Concrete":
        workability = st.select_slider("Concrete workability", options=["Low", "Medium", "High"], value="Medium")
        nm_size = st.selectbox("Nominal maximum aggregate size", [9.5, 12.5, 19.0, 25.0], format_func=lambda x: f"{x:g} mm")
        air_content = st.number_input("Assumed entrapped air (%)", min_value=0.0, max_value=5.0, value=2.0, step=0.1)
        dry_rodded_uw = st.number_input(
            "Dry-rodded unit weight of coarse aggregate (kg/m³)",
            min_value=1200.0,
            max_value=1900.0,
            value=1600.0,
            step=25.0,
        )
    else:
        workability = st.select_slider("Mortar workability", options=["Low", "Medium", "High"], value="Medium")
        sand_ratio = st.selectbox("Sand-to-binder ratio by volume", [3.0, 4.0, 5.0, 6.0], index=1)
        air_content = st.number_input("Assumed entrapped air (%)", min_value=0.0, max_value=5.0, value=2.0, step=0.1)

    st.markdown("### Binder composition")
    c1, c2 = st.columns(2)
    with c1:
        slag_pct = st.slider("Slag (%)", 0, 30, 15)
    with c2:
        mk_pct = st.slider("Metakaolin (%)", 0, 30, 5)

    opc_pct = 100 - slag_pct - mk_pct
    st.metric("OPC (%)", f"{opc_pct}%")

    if opc_pct < 0:
        st.error("Binder percentages exceed 100%. Reduce slag and/or metakaolin.")

    if wb < 0.40:
        sp_dosage_pct = st.slider("SP dosage (% by total binder mass)", 0.5, 3.0, 0.75, 0.05)
        st.caption("Typical starter range is 0.5% to 1.0%. Some high-performance systems use higher dosages.")
    else:
        sp_dosage_pct = 0.0

    generate = st.button("Generate mix", type="primary")

with col2:
    st.subheader("Design checks")
    binder_pct = {"OPC": opc_pct, "Slag": slag_pct, "Metakaolin": mk_pct}

    warnings: List[str] = []
    if sum(binder_pct.values()) != 100:
        warnings.append("Binder percentages must sum to 100%.")
    if slag_pct > 30:
        warnings.append("Slag exceeds the 30% max limit.")
    if mk_pct > 30:
        warnings.append("Metakaolin exceeds the 30% max limit.")
    if wb < 0.35 or wb > 0.60:
        warnings.append("w/b is outside the current app bounds.")

    if mix_type == "Concrete":
        recommended_wb = recommended_wb_concrete(target_strength)
    else:
        recommended_wb = recommended_wb_mortar(target_strength)

    st.metric("Recommended max w/b for target strength (starter)", f"{recommended_wb:.2f}")
    if wb > recommended_wb:
        warnings.append(
            f"Selected w/b ({wb:.2f}) is above the starter recommended maximum ({recommended_wb:.2f}) for the chosen target strength."
        )

    if wb < 0.40:
        warnings.append("w/b below 0.40: include superplasticizer and verify segregation/bleeding risk in trial batches.")

    if mix_type == "Concrete":
        st.caption("Concrete uses an ACI-inspired absolute-volume path with workability, air, and nominal max aggregate size.")
    else:
        st.caption("Mortar uses workability, w/b, and selected sand-to-binder ratio by volume.")

    for w in warnings:
        st.warning(w)

if generate and opc_pct >= 0:
    if mix_type == "Concrete":
        results = design_concrete(
            target_strength_mpa=target_strength,
            wb=wb,
            workability=workability,
            nm_size_mm=nm_size,
            batch_m3=batch_m3,
            binder_pct=binder_pct,
            props_df=props_df,
            dry_rodded_unit_weight_kg_m3=dry_rodded_uw,
            air_content_pct=air_content,
            sp_dosage_pct=sp_dosage_pct,
        )
    else:
        results = design_mortar(
            target_strength_mpa=target_strength,
            wb=wb,
            workability=workability,
            batch_m3=batch_m3,
            binder_pct=binder_pct,
            props_df=props_df,
            sand_to_binder_vol_ratio=sand_ratio,
            air_content_pct=air_content,
            sp_dosage_pct=sp_dosage_pct,
        )

    totals = totals_from_results(results, props_df)
    out_df = build_output_table(results, props_df, batch_m3, unit_system)

    st.markdown("---")
    st.subheader("Results")
    m1, m2, m3 = st.columns(3)
    with m1:
        st.metric("Total batch cost", f"${totals['cost_usd']:,.2f}")
    with m2:
        st.metric("Total embodied carbon", f"{totals['carbon_kgco2e']:,.1f} kg CO₂e")
    with m3:
        st.metric("Batch volume", f"{batch_volume_input:,.2f} {batch_volume_label(unit_system)}")

    st.dataframe(out_df, use_container_width=True, hide_index=True)

    csv = out_df.to_csv(index=False).encode("utf-8")
    st.download_button(
        label="Download results as CSV",
        data=csv,
        file_name="mix_design_results.csv",
        mime="text/csv",
    )

    st.markdown("### Notes")
    st.markdown(
        """
- This is an MVP and uses editable starter assumptions.
- Concrete sizing follows a simplified ACI-inspired absolute-volume workflow.
- Mortar sizing uses workability plus a sand-to-binder volume ratio.
- Cost and carbon factors are starter defaults; replace them with your supplier quotes and EPD data.
- Final acceptance should always come from trial batching and testing.
"""
    )
else:
    st.info("Set your inputs and click **Generate mix**.")
