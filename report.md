# Dynamic Parking Pricing Models Report

## Introduction/Project Overview

This report details the development and evaluation of different dynamic pricing models for urban parking lots. The core **goal** is to implement and compare models that adjust parking prices in real-time based on various factors. By dynamically adjusting prices based on demand and other relevant conditions, the aim is to optimize parking lot **occupancy** and **revenue**, providing a more efficient parking system for both operators and users. The project employs a **data-driven approach**, simulating real-time scenarios to test the effectiveness of each pricing strategy.

## Data Processing Steps

A custom `ParkingDataProcessor` class (defined in **Cell 2**) was used to load, clean, and prepare the raw parking data (`dataset.csv`). This class encapsulates the essential steps required to transform the raw data into a format suitable for the pricing models. Key preprocessing steps included:

1.  **Creating a `DateTime` column:** Combining the `LastUpdatedDate` and `LastUpdatedTime` columns into a single, properly formatted datetime object. This is crucial for analyzing time-series data and simulating real-time updates chronologically.
2.  **Calculating `OccupancyRate`:** Computing the ratio of `Occupancy` to `Capacity` for each record. This provides a standardized metric (ranging from 0 to 1 or slightly above if over capacity) that is a primary indicator of parking demand.
3.  **Encoding Categorical Variables:** Converting the `VehicleType` and `TrafficConditionNearby` columns into numerical representations (`VehicleType_encoded`, `TrafficCondition_encoded`). This allows these qualitative features to be used in mathematical models, assigning weights based on their assumed relative impact on demand (e.g., trucks might take more space or time, high traffic might deter drivers).
4.  **Extracting Time Features:** Deriving `Hour` and `DayOfWeek` from the `DateTime` column. These features capture temporal patterns in parking demand, as needs and traffic conditions often vary significantly by hour of the day and day of the week.

These steps ensure the data is clean, consistent, and contains the necessary features for the pricing models to function effectively.

## Model Descriptions

Three different dynamic pricing models were implemented and tested:

### Baseline Linear Model

This is the simplest model (implemented in **Cell 3**). It's a basic **reactive pricing** strategy. The price is adjusted solely based on the `OccupancyRate`. The core logic is a linear relationship: `Price(t+1) = Price(t) + α * OccupancyRate`. The `alpha` parameter (sensitivity) determines how aggressively the price changes in response to occupancy. A higher `alpha` means prices will increase or decrease more sharply with changes in occupancy. The model also includes **price bounds** to prevent extreme values, ensuring the price stays within a defined range (0.5x to 2x the `base_price`).

### Demand-Based Model

This model (implemented in **Cell 4**) adopts a more sophisticated **demand-responsive pricing** approach. Instead of relying only on occupancy, it calculates a composite **'raw demand' score** based on a weighted linear combination of multiple factors:
-   `OccupancyRate` (weight `alpha`)
-   `QueueLength` (weight `beta`)
-   `TrafficCondition_encoded` (weight `gamma`)
-   `IsSpecialDay` (weight `delta`)
-   `VehicleType_encoded` (weight `epsilon`)

The weights (`alpha`, `beta`, `gamma`, `delta`, `epsilon`) are parameters that can be tuned to reflect the perceived importance of each factor in driving demand. A negative weight is applied to `TrafficCondition_encoded` because high traffic is assumed to *decrease* the desirability (and thus potentially the effective demand) for parking nearby due to difficulty in access.

The raw demand score is then **normalized** using the hyperbolic tangent (`tanh`) function. This squashes the raw demand value into a range between -1 and 1, preventing the price calculation from becoming overly sensitive to extreme raw demand values. The final price is calculated as `Base Price * (1 + lambda * Normalized Demand)`, where `lambda` is a **price sensitivity** parameter controlling how much the price reacts to the normalized demand. This model also enforces the same **price bounds** as the baseline model.

### Competitive Pricing Model

This model (implemented in **Cell 5**) extends the **demand-based approach** by incorporating the concept of **competitive pricing**. It recognizes that parking lots don't operate in isolation and that nearby competitors' prices can influence customer choice and thus, the optimal price.

It first calculates a demand-based price similar to the Demand-Based Model. However, it then **finds nearby competitors** by checking the `Latitude` and `Longitude` of other parking lots within a specified `proximity_threshold` (using a simplified Euclidean distance). It calculates the **average price of these nearby competitors**, potentially using a weighted average where closer competitors have more influence.

The final price is a **blend** of the demand-based price and the competitor average price. The `competition_weight` parameter determines the balance; a higher weight means the model is more influenced by competitor pricing, while a lower weight means it relies more on its internal demand calculation. The logic involves a slight adjustment to the raw demand based on whether the competitor average price is higher or lower than the base price, before blending the resulting demand-based price with the competitor average price. This model also includes the same **price bounds**.

## Demand Function Explanation

In the **Demand-Based** and **Competitive Pricing Models**, the core of the pricing logic is the calculation of a **raw demand score**. This is achieved through a **weighted linear combination** of several input features:

`Raw Demand = (alpha * Occupancy Rate) + (beta * Queue Length) - (gamma * Traffic Condition) + (delta * Is Special Day) + (epsilon * Vehicle Type Weight)`

-   **`alpha` (Occupancy Rate):** A positive weight. Higher occupancy suggests higher current demand or desirability, increasing the raw demand score.
-   **`beta` (Queue Length):** A positive weight. A longer queue indicates strong immediate demand exceeding current capacity, significantly increasing the raw demand score.
-   **`gamma` (Traffic Condition):** A positive weight, but applied with a *negative* sign in the sum. Indicates encoded traffic condition (e.g., low, average, high). High traffic nearby makes the lot harder to access, potentially reducing effective demand. The negative sign ensures higher traffic scores *decrease* the raw demand.
-   **`delta` (Is Special Day):** A positive weight. Special days (like holidays or events) are assumed to increase overall demand for parking, adding to the raw demand score.
-   **`epsilon` (Vehicle Type Weight):** A positive weight. Different vehicle types might have different impacts on lot turnover or space utilization, influencing the demand calculation. The encoded weights reflect these assumed differences.

The resulting `Raw Demand` score is a single number intended to capture the aggregate pressure on the parking lot at a given moment.

This raw score is then passed through a **normalization step** using the `tanh(x / scale_factor)` function. Normalization is essential because the raw demand can vary significantly depending on the input values and weights. `tanh` squashes any input value into the range (-1, 1). This prevents extremely high or low raw demand scores from resulting in unboundedly high or low price multipliers, ensuring the final price remains within a predictable and bounded range, making the model more stable and preventing unrealistic price predictions.

## Assumptions Made

Several assumptions underpin the design and simulation of these models:

-   **Data Accuracy:** It is assumed that the provided `dataset.csv` accurately captures real-world parking conditions, including occupancy, queue length, traffic, and special day information, without significant errors or biases.
-   **Encoded Value Representation:** The numerical values assigned during categorical encoding (`VehicleType_encoded`: car=1.0, bike=0.5, truck=1.5, cycle=0.3; `TrafficCondition_encoded`: low=0.5, average=1.0, high=1.5) are assumed to accurately reflect the *relative impact* of these categories on parking demand. For example, a truck is assumed to contribute more to "demand pressure" (perhaps due to space or maneuverability) than a car, and high traffic is assumed to have a stronger negative impact than low traffic.
-   **Parameter Appropriateness:** The specific numerical values chosen for the model parameters (`alpha`, `beta`, `gamma`, `delta`, `epsilon`, `lambda`, `competition_weight`, `proximity_threshold`) are assumed to be reasonable starting points or approximations of how these factors *should* influence pricing in a real urban environment. In a production system, these parameters would ideally be learned from historical data or expert knowledge.
-   **Linear Relationships:** The demand function assumes a primarily **linear relationship** between the weighted sum of features and the raw demand. While normalized later, the initial aggregation is linear. This is a simplification; real-world demand might have more complex, non-linear relationships with these factors.
-   **Simplified Competition:** The Competitive Pricing Model uses a simplified distance calculation and assumes that competitor influence can be captured by a weighted average of nearby prices within a fixed radius and a simple binary adjustment to raw demand based on whether competitor prices are above the base price. Real competition might involve more complex dynamics and market segmentation.
-   **Real-time Simulation Fidelity:** The simulation engine processes records sequentially by timestamp, mimicking a real-time data stream. It assumes that the model's decision at one timestamp immediately influences the 'state' (current price) for the next relevant record, regardless of the actual time gap.

## Price Changes with Demand and Competition

The way price changes in each model directly reflects its underlying logic:

-   **Baseline Linear Model:** Price changes are **directly and linearly proportional** to the **Occupancy Rate**. If occupancy increases, price increases; if occupancy decreases, price decreases. The magnitude of this change is scaled by the `alpha` parameter. It does *not* consider queue length, traffic, special days, vehicle types, or competitor prices.
-   **Demand-Based Model:** Price changes are driven by the **composite Normalized Demand score**. As the calculated raw demand (influenced by all five factors: occupancy, queue, traffic, special day, vehicle type) increases, the normalized demand increases (towards 1), leading to a higher price. Conversely, as raw demand decreases, normalized demand decreases (towards -1), leading to a lower price. The `lambda` parameter controls the sensitivity of price to this normalized demand. This model is more responsive to a broader set of conditions than the baseline.
-   **Competitive Pricing Model:** Price changes are influenced by **both the calculated demand factors AND the prices of nearby competitors**. The model first calculates a demand-based price. It then considers the average competitor price. The final price is a weighted average of these two values.
    -   If the demand factors suggest a high price, but competitors are priced significantly lower, the `competition_weight` will pull the final price downwards to remain competitive.
    -   If demand is low, but competitors are priced higher, the `competition_weight` might pull the price upwards slightly compared to a pure demand-based approach, potentially allowing the lot to capture some demand from the higher-priced competitors.
    -   The model dynamically balances responding to local conditions (demand) with reacting to the external market (competitors).

## Model Testing and Comparison

Each of the three models (`BaselineLinearModel`, `DemandBasedModel`, `CompetitivePricingModel`) was tested using the `RealTimeSimulator` (defined in **Cell 6**). To facilitate quicker testing and visualization, the simulation was run on a **subset** of the processed data (the first 500 records in the executed cells). This simulation mimics a real-time scenario by processing data records chronologically.

The simulation results for each model are stored in separate DataFrames (`results1`, `results2`, `results3`). These DataFrames contain the predicted price at each timestamp, along with the input features and, for the more complex models, intermediate calculations like raw/normalized demand and competitor price.

The **comparison** focuses on analyzing the characteristics of the prices generated by each model, including:
-   Overall **Average Price** and **Price Range** observed during the simulation (`comparison_df`).
-   How prices **fluctuate over time** (visualized price trends).
-   The **relationship** between key input features (like occupancy or demand) and the predicted price.
-   The **distribution** of prices generated by each model.

By comparing these aspects across the three models using the simulation results and the `comparison_df`, we can evaluate their respective behaviors and potential strengths and weaknesses in a dynamic environment.

## Visualizations

The `BokehPricingVisualizer` class (defined in **Cell 7**) is used to generate interactive plots that help understand the models' behavior:

-   **Price Trends Over Time:** Plots showing how the predicted price for selected parking lots (or the average price across all lots for comparison) changes over the sequence of records processed. This helps visualize price volatility, responsiveness, and overall price levels for each model.
-   **Occupancy Rate vs Price:** A scatter plot showing the relationship between the occupancy rate and the predicted price. This highlights how directly and strongly each model links pricing to this key metric. The trend line provides a summary of this relationship.
-   **Demand Analysis (Raw/Normalized Demand vs Price):** Scatter plots (for Demand-Based and Competitive models) showing how the calculated raw and normalized demand scores relate to the predicted price. This illustrates the effectiveness of the demand function and normalization in driving price adjustments.
-   **Summary Statistics Plots:** A grid of plots providing an overview:
    -   A histogram of the price distribution shows the range and frequency of different price points generated by the model.
    -   Plots of average price by vehicle type and traffic condition show how these categorical factors implicitly (via the demand function) or explicitly influence pricing decisions on average.
    -   A scatter plot of Queue Length vs Price shows the model's responsiveness to queue size.
-   **Model Comparison - Price Trends:** A line plot comparing the average price across all lots for each model over the simulation records. This provides a high-level view of how the overall price level evolves differently under each strategy.

These visualizations are crucial for qualitatively assessing the models' behavior and comparing their dynamic responses to changing parking conditions.

## Conclusion

Based on the simulation results, the price trends observed, the relationship between price and influencing factors, and the summary statistics presented in the visualizations, this section will summarize the key findings for each model. It will discuss which model appears most promising for dynamic parking pricing based on the simulated data and the defined objectives, and outline potential future work, such as parameter tuning, incorporating more data features, or exploring more complex modeling techniques.
