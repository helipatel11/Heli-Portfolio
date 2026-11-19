---
title: "Plant Pal: My First MLOps Pipeline"
date: 2025-11-16
weight: 20
description: "An early project: a healthy/unhealthy leaf classifier with MobileNetV2, a ZenML pipeline, MLflow and a Streamlit app, and what I'd do differently now."
tags: ["mlops", "zenml", "mlflow", "tensorflow", "streamlit", "earlier-work"]
---

*Built in November 2025, before my [LLMOps Support Assistant](/projects/llmops-support-assistant/). I've kept it here as a record of where I started.*

## What it is

Plant Pal classifies a leaf photo as **healthy** or **unhealthy**. It was my first attempt at wiring a model into an MLOps toolchain rather than training it in a notebook. The goal was to learn the toolchain end to end, not to build an accurate model, so I used a small sample dataset:

- **Model:** MobileNetV2 pre-trained on ImageNet, with its base frozen and a single sigmoid output trained on top (transfer learning).
- **Pipeline:** a ZenML pipeline with three steps: load data → train → evaluate.
- **Tracking:** MLflow logs the run and the saved model artifact.
- **App:** a local Streamlit app where you upload an image and get a prediction, plus a feedback form that saves the image and label to a local folder for future retraining.

![Plant Pal architecture](/images/plant-pal-architecture.png)

Everything runs locally. Nothing is deployed to the cloud.

## What I learned

- **How a pipeline tool structures ML work.** ZenML made me split the work into steps with explicit inputs and outputs, instead of one long script.
- **What experiment tracking is for.** MLflow gave each run an ID and kept the model artifact with it.
- **Transfer learning on a small dataset.** Freezing a pre-trained network and training only the last layer is a sensible way to start when you have few images.
- **Closing the loop.** The feedback form in the app was my first attempt at collecting real-world labels to improve a model over time.

## Looking back (November 2026)

Looking at it now, the main thing I'd change is in the pipeline's data step:

```python
X_train, y_train = next(train_gen)
X_val, y_val = next(val_gen)
```

`next()` returns **one batch**, so with `batch_size=32` the model would never see more than 32 training images, however large the dataset. My sample dataset had only 16 images, so it made no difference and didn't show up at the time. The fix is to pass the full generators to training and evaluation.

Other things I'd do differently:

- **Make the evaluate step meaningful:** a held-out test set and per-class precision, recall and a confusion matrix, so the pipeline can tell a good model from a bad one.
- **Track training properly.** MLflow logged the saved model file, but no training metrics or parameters, so runs couldn't be compared.
- **Pin dependencies** so the project can be reproduced.

My [LLMOps Support Assistant](/projects/llmops-support-assistant/) is what that looks like in practice: held-out evaluation, versioned components, and every result recorded with how and when it was measured.

**Code:** [github.com/helipatel11/plant-pal-mlops](https://github.com/helipatel11/plant-pal-mlops)
