---
title: Concept Drift Detection
layout: page
nav_order: 3
---

# What is concept drift and why is it important to detect?
{: .fs-9 }

In a streaming setting, the relationship between the predictor attributes and the target attribute is dynamic. This relationship between the attributes can be called a concept. The change in the relationship over time can lead to a predictive model trained on previous data to make less accurate predictions. Such a change in the relationship can be considered a deviation or "drift" in the concept. Concept drift can cause losses due to the deterioration of the model's predictive performance, in addition to the resources required to retrain the model to adapt to the new concept. Hence, the timely detection of such a drift in the concept is extremely valuable in a dynamic setting to preserve resources.

Take the example of a regression model used to predict the number of cycles expected to be rented out during any given day of the year. The first scenario is to use a static model i.e., it is trained using a bulk of the data once and then deployed to make predictions on incoming streaming data. This model is trained in an offline manner and tested or used in an online fashion. Let us assume that the model has great accuracy, but then starts showing a decline in the predictive performance in the year 2020. Upon inspection, it is found that due to then ongoing COVID crisis, the number of cycles rented on a given day were much less compared to a similar day the previous year. The model trained on last year's data was hence providing an overestimate of the number of rentals. This overestimation could potentially cause over-staffing and excess costs to the businesses relying on the model's predictions. This cost is in addition to the cost of retraining the model on the data for the COVID period. Furthermore, the model will then underestimate the number of rentals on a given day in 2021, since it was trained on data from the COVID period. This problem would not only occur during such extreme changes in the data environment, but can also occur due to seasonal variation in the data.

Now, consider a scenario where the same model is retrained on new data periodically. Let us assume the period of retraining is a month. This would mean that the model will have the latest data to train on and will give better short-term predictions. However, with the volume of the data used for training being limited to the most recent month's worth, the model runs the risk of overfitting on the data. More importantly, the cost of training the model repeatedly on the monthly data will stack up. Using the same example that we did in the previous scenario, the number of bike rentals was only affected by COVID in 2020. This model would perform well throughout the crisis, since it will base its predictions on the most recent trends in the data. However, in 2021, the model might begin underperforming again. While the most recent trends can be captured by the retraining, the overall trends observed in previous years when there were no crises will not be taken into account by this model. This could lead to discrepancy in the results. Moreover, in a normal year, the patterns observed in the previous years should match the current pattern, which would imply that there is no requirement to retrain the model so frequently.

Finally, in the same scenario, let us assume that the learning pipeline incorporates a concept drift detection method such that the retraining of the model is triggered when concept drift is detected. This pipeline ensures that the latest "drifted" data is used to train the model and the short-term trends will be captured by the model. In the COVID crisis scenario, the model could detect a concept drift in the data, owing to the changed environment, and then trigger the retraining of the model in 2020. This will allow the model to regain its performance after a short period of decline observed before the concept drift is detected. Further down the line in the scenario, in 2021, when the COVID crisis is uplifted and bike rentals gradually return to the prior years' values, the model will again adapt to the changed data environment following a short period of decline before the concept drift is detected.