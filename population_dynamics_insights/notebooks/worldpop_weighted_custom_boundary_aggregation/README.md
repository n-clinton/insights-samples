# Population Dynamics Insights: WorldPop-Weighted Custom Boundary Aggregation

> **⚠️ Important Requirement:** To run the queries in this notebook, your Google Cloud Project must have access to the **US Population Dynamics Insights dataset**. For instructions on how to request and configure access, see [Set up Population Dynamics Insights](https://developers.google.com/maps/documentation/population-dynamics-insights/cloud-setup).

### Overall Goal

This guide demonstrates how to use **Population Dynamics Insights (PDI)** alongside **Google Earth Engine (GEE)** to perform **Custom Boundary Aggregation**: taking native S2 cell (Level 12) embeddings and accurately rolling them up into custom, arbitrary polygons (like drive-time isochrones or sales territories) using a highly precise **Population-Weighted Average**.

**The Scenario:** You want to use PDI's 330-dimensional embeddings as features in an ML model predicting retail store performance. Your target geographic boundaries are 5km radii. Because human activity is rarely spread evenly across physical space, a naive "area-weighted" aggregation (assuming a 50% overlap equals 50% of the signal) will warp your ML features. By performing all calculations directly in BigQuery, we leverage high-resolution raster population data, specifically the [WorldPop USA 2025 Population Counts (100m resolution) dataset](https://hub.worldpop.org/geodata/summary?id=75983) (see attribution and licensing at the end of this notebook), to properly weight each intersecting S2 "sliver" based on actual human density.

*🌟 Note on Temporal Alignment: We explicitly align our 2025 PDI Embeddings with the 2025 WorldPop demographic dataset to avoid temporal confounding, a critical best practice in spatial machine learning!*

### Key Technologies Used

*   **[Population Dynamics Insights](https://developers.google.com/maps/documentation/population-dynamics-insights/overview):** To provide the underlying 330-dimensional embeddings capturing geographic, environmental, and map features.
*   **[BigQuery GIS](https://cloud.google.com/bigquery):** To execute spatial overlays (`ST_INTERSECTS`, `ST_INTERSECTION`) and math natively in the data warehouse.
*   **[Google Earth Engine (GEE)](https://earthengine.google.com/):** Used natively within BigQuery via `ST_REGIONSTATS` to access the [WorldPop 2025 Population Counts dataset](https://hub.worldpop.org/geodata/summary?id=75983) without any raster processing overhead.
*   **[CARTO Analytics Toolbox](https://docs.carto.com/data-and-analysis/analytics-toolbox-overview):** To dynamically generate S2 boundary polygons from PDI string tokens.
*   **Python Libraries:** **[Pandas](https://pandas.pydata.org/)** (data manipulation) and **[NumPy](https://numpy.org/)** (mathematical validation).

*Note: This notebook executes queries that incur BigQuery costs. See [BigQuery Pricing](https://cloud.google.com/bigquery/pricing) for details.*

### The Step-by-Step Workflow

1.  **Generate Target Boundaries:** We use BigQuery GIS to construct dummy 5km store isochrones around major US cities to act as our custom boundaries.
2.  **Calculate Total Populations:** We use Earth Engine to compute the total human population residing inside each 5km boundary.
3.  **Intersect & Calculate Slivers:** We load the PDI data, join the S2 cells against our store boundaries, cut out the precise overlapping spatial "slivers", and use Earth Engine again to determine the exact population living within that specific sliver.
4.  **Apply Weights:** We multiply the 330 PDI dimensions by their population weight ratio and sum the vectors natively in BigQuery.
5.  **Optional Python Normalization:** We demonstrate how to apply L2 Renormalization using `scikit-learn` for users whose models require strictly normalized unit vectors.
6.  **Mathematical Validation:** We extract the final array into NumPy to programmatically prove it retains 330 dimensions and a magnitude of `1.0`.

### How to Use This Notebook

1.  **Prerequisites & Secrets:** Before running this notebook, you must:
    *   Enable the **BigQuery API** and the **Google Earth Engine API** in your Google Cloud Project.
    *   Configure an environment variable in the Colab "Secrets" tab (the **key icon** on the left menu): `GCP_PROJECT_ID`. This should be your Google Cloud Project ID. **Crucially, this project must be authorized to access the US Population Dynamics Insights dataset.**
2.  **Authentication:** The first code cell will prompt you to authenticate your Google Account. Ensure the account you use has BigQuery Data Viewer/Job User permissions and Earth Engine Resource Viewer permissions for your Project ID.
3.  **Run the Cells:** Once authenticated, execute the cells in order from top to bottom.