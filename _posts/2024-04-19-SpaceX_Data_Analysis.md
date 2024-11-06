---
layout: post
title: SpaceX Falcon 9 Landing Success Prediction - A Data Science Deep Dive
categories: [Data Science, Machine Learning, Python, SQL]
---
SpaceX's ability to reuse first-stage boosters has revolutionized space launch economics, reducing costs from $165+ million to $62 million per launch. This analysis combines multiple data science techniques to predict landing success:

## Executive Summary


- Data collection via REST API and web scraping
- Statistical modeling and machine learning
- Interactive data visualization
- SQL analysis
- Geographic mapping

## Methodology

### Data Collection Pipeline

1. **REST API Collection**:
   - Accessed SpaceX API for launch data
   - Converted JSON responses to pandas DataFrames
   - Filtered for Falcon 9 launches only

2. **Web Scraping**:
   ```python
   # Creating BeautifulSoup object
   soup = BeautifulSoup(data.text, 'html.parser')
   
   # Getting column names
   column_names = []
   for column in soup.find_all('th'):
       column_names.append(column.text.strip())
   ```

3. **Data Wrangling**:
   ```python
   # Create landing outcomes
   landing_outcomes = df['Outcome'].value_counts()
   bad_outcomes = set(landing_outcomes.keys()[[1,3,5,6,7]])
   
   # Binary classification
   df['Class'] = df['Outcome'].apply(
       lambda x: 0 if x in bad_outcomes else 1)
   ```

## Exploratory Data Analysis

### Launch Site Analysis

SQL query for launch site success rates:
```sql
SELECT LaunchSite, 
       COUNT(*) as total_launches,
       SUM(Class) as successful_landings,
       CAST(SUM(Class) AS FLOAT) / COUNT(*) as success_rate
FROM SPACEXDATASET
GROUP BY LaunchSite
ORDER BY success_rate DESC;
```

Results:
- KSC LC-39A: 76.9% success rate (highest)
- CCAFS LC-40: 29.2%
- VAFB SLC-4E: 16.7%
- CCAFS SLC-40: 12.5%

### Payload Analysis

Key findings:
1. Payload Range 0-5000kg:
   - More diverse outcomes
   - Higher success rate variability

2. Payload Range 6000-10000kg:
   - More consistent outcomes
   - Lower success rate overall

### Temporal Analysis

Success rate trends:
- Significant increase since 2013
- Minor drop in 2018
- Strong recovery post-2018

## Geographic Analysis with Folium

Created interactive maps using Folium:

```python
def add_site_marker(name, coordinate, success_rate):
    color = get_success_color(success_rate)
    folium.Circle(
        coordinate,
        radius=1000,
        color=color,
        popup=f"{name}\nSuccess Rate: {success_rate}%"
    ).add_to(site_map)
```

Key findings:
1. Launch Site Locations:
   - Concentrated in Florida and California
   - All sites near coastlines
   - Strategic proximity to infrastructure

2. Proximity Analysis:
   - Railway access within reasonable distance
   - Clear flight paths over water
   - Safe distance from population centers

## Machine Learning Results

### Model Performance Comparison

1. **Decision Tree** (Best Performer):
   - Accuracy: 89%
   - Key parameters:
     ```python
     {'criterion': 'gini',
      'max_depth': 6,
      'max_features': 'auto',
      'min_samples_leaf': 2}
     ```

2. **Support Vector Machine**:
   - Accuracy: 84%
   - Parameters:
     ```python
     {'C': 1.0,
      'gamma': 0.0316,
      'kernel': 'sigmoid'}
     ```

3. **KNN**:
   - Accuracy: 84%
   - Parameters: n_neighbors=10

4. **Logistic Regression**:
   - Accuracy: 84%
   - Parameters: C=0.01

### Confusion Matrix Analysis

Performance metrics:
- Sensitivity: 1.00 (TPR = TP/(TP + FN))
- Specificity: 0.50 (SPC = TN/(FP + TN))
- Precision: 0.80 (PPV = TP/(TP + FP))
- F1 Score: 0.89 (2TP/(2TP + FP + FN))
- False Positive Rate: 0.50 (FP/(FP + TN))

## Interactive Dashboard

Created a Plotly Dash dashboard with:
1. Launch site selection dropdown
2. Payload mass range slider
3. Success rate visualizations
4. Mission outcome statistics

```python
@app.callback(
    Output('success-pie-chart', 'figure'),
    [Input('site-dropdown', 'value'),
     Input('payload-slider', 'value')])
def update_graph(site, payload_range):
    filtered_df = filter_data(site, payload_range)
    fig = create_success_chart(filtered_df)
    return fig
```

## Key Findings

1. **Launch Site Performance**:
   - KSC LC-39A emerged as the most successful site
   - Coastal locations show better success rates
   - Infrastructure proximity correlates with success

2. **Payload Impacts**:
   - Optimal range: 0-5000kg
   - Higher masses require more precise conditions
   - Success rates vary by orbit type

3. **Technical Evolution**:
   - Clear learning curve in success rates
   - Technology improvements visible in data
   - Recent launches show highest reliability

## Future Improvements

1. **Enhanced Data Collection**:
   - Real-time weather data integration
   - More detailed technical specifications
   - Extended historical data

2. **Model Refinements**:
   - Ensemble methods exploration
   - Deep learning implementation
   - Feature engineering optimization

3. **Visualization Upgrades**:
   - Real-time success prediction
   - 3D trajectory visualization
   - Interactive failure analysis

## Resources and References

- [SpaceX API Documentation](https://github.com/r-spacex/SpaceX-API)
- [Project GitHub Repository](https://github.com/TheCode2019/IBM_powerpoint)

This analysis provides valuable insights for space launch companies and demonstrates the power of data science in modern space operations. The high accuracy of our predictive models shows the potential for data-driven decision-making in space launch planning.

