# Project Spec Template (Design-Only)
## Skytrain Crowding Predictor

## 1) User & Decision
- User: Skytrain operations planner or passenger
- Decision: Adjust train frequency (operations) or choose less crowded trains (passengers)
    
## 2) Target & Horizon
- Target: Predicted % of train occupancy (continuous/numeric, from 0-100% capacity)
- Horizon: The user-specified train arrival time
## 3) Features (No Leakage)
- Train stop 
- Train direction
- Time features: time of day, day of week, day of year 
- Historical average occupancy at that stop/time
Exclude
- Any future occupancy data that occurs after predictions from historical data (avoid leakage)
- PII (Personally Identifiable Information) and sensitive operational info (eg. PII of employees, accident incident reports)
## 4) Baseline → Model Plan
- Baseline rule: Predict the overall mean occupancy across all trains, stops, and times in the historical dataset, and at prediction time, always return this single value, regardless of train, stop, direction, or time. This global average is simple, fast, and easy to implement, but ignores any variation due to station, time, direction, or calendar events.
- Simple model: A gradient boosted tree-based model like LightGBM (Light Gradient-Boosting Machine, a fast, efficient tree-based machine learning algorithm optimized for large datasets and low-latency predictions) will outperform the baseline because it can capture interactions between features like time of day, train stop, direction, and historical patterns, rather than assuming everything is just the average. LightGBM only requires low memory usage and is quickly trained, even on large datasets.
## 5) Metrics
- MAE (Mean Absolute Error,  a metric that measures the average of the absolute differences between a model's predicted values and the true (actual) values) percentage points of occupancy is a viable metric  that can be used as it is easy to explain and interpret. This is good for giving passengers or planners a sense of average reliability, but not sufficient alone if missing spikes in occupancy creates overcrowding harms.
- AUC-PR (Area Under the Precision-Recall Curve measures how well a model distinguishes positives from negatives, especially useful when the dataset is imbalanced)can be used as a supplementary metric since it is more informative (in comparison to other metrics) with imbalanced data and thus best for minimizing passenger harm from missed crowded-train warnings (false negatives) while balancing annoyance from false alarms (false positives).

## 6) API Sketch
POST /predictCrowding
Request body
{
  "train_stop": "Waterfront",
  "train_direction": "Northbound",
  "datetime": "2025-09-14T08:30:00"
}
Response body
{
  "predicted_occupancy_pct": 72,
  "confidence": 0.85
}

## 7) Privacy, Ethics, Reciprocity (PIA excerpt)
- Data inventory: aggregated train occupancy counts (per stop, direction, and time bucket); no PII (names, device IDs, etc).
Purpose: predict future train crowding to inform passengers and operations planning.
Retention: raw logs (e.g., per-trip counts) retained ≤ 14 days; aggregated features stored 90–180 days for modeling.
- Access: role-based least privilege; engineers and data scientists only, no external backdoors.

- Telemetry decision matrix:

Train stop/direction metadata → high value, low invasiveness, low effort → keep

Time of day/day of week → high value, low invasiveness, low effort → keep

Aggregate occupancy counts → high value, low invasiveness, medium effort → keep

Exact geolocation of individuals → low value, high invasiveness, high effort → drop

- Guardrails: enforce k-anonymity on occupancy counts; apply jitter/aggregation to prevent re-identification; disclose telemetry collection in documentation; support opt-in where applicable.

- Reciprocity: insights are returned to passengers (crowding predictions for better travel planning) and to operations planners (capacity management and scheduling).
## 8) Architecture Sketch (1 diagram)
Major components and data flow. Note trade-offs and alternatives.
						
## 9) Risks & Mitigations
1. Risk: Collecting identifiable passenger data (e.g., ticket IDs, device IDs, precise geolocation) could expose personal information.
   Mitigation: Excluded all PII from features, collected train stop and direction instead of tracking user's live geographical location. Only aggregate counts/time/location buckets are used.
3. Risk:  Model bias across routes/times. Some routes/times may be underrepresented in training data, leading to biased crowding predictions.
   Mitigation: Monitor per-route and per-time-slot error rates.
   How to test: Slice evaluation (e.g., MAE by route, by peak/off-peak).
5. Risk: reliability under surge traffic -- API traffic spikes (50k/hour) can lead to latency or failures.
   Mitigation: Deploy with AWS Lambda and API Gateway auto-scaling, add caching tier for frequent queries.
   How to test: Load test with Locust or JMeter at surge levels, track p95 latency under stress.
## 10) Measurement Plan
Metric: Compare Mean Absolute Error (MAE) between baseline dummy model and the trained LightGBM model. HTe trained LightGBM model should reduce MAE by ≥20% over baseline.
- p95 Latency: <500ms per request. This will ensure predictions are fast enough to be served in real time within a transit app, where users expect crowding information before boarding.
- Max Cost per 10k Predictions: <$1. LightGBM is lightweight and runs efficiently on AWS Lambda (~100ms execution, 128–256MB). Lambda offers automatic scaling for surges without pre-provisioning, with API Gateway and CloudWatch integration. While GCP and Azure offer similar options, AWS’s breadth of cloud services, proven reliability, tooling, and scalability make it the most reliable choice for real-time transit predictions.

## 11) Evolution & Evidence	
Link a git hash (or range/tag) that shows the design’s evolution (commits, README updates, diagrams).
Insight memo link (3 insights), assumption audit, and Socratic log references.
						
					


Make sure to address:	
Target & Metrics: No leakage; harms-aware metric; SLA and cost budget			
Privacy: k-anonymity suppression, jitter/aggregation, retention, access boundaries, reci- procity.
Compute: Serverless vs. containers vs. hybrid with cost and scaling trade-offs. 4	
Traffic: Rate limits, caching tiers; degradation modes under surge.
Auth: API keys/OAuth/IAM; align with privacy and cost.
Observability: Minimal telemetry consistent with PIA. 
