# Complete Guide: Local AI/ML Models - Running, Fine-tuning & Deployment

## Part 1: Running Models Locally

### Option A: Running LLMs (Large Language Models)

#### 1. **Ollama** (Easiest for beginners)
```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Run a model
ollama run llama3.2
ollama run mistral
ollama run codellama

# List available models
ollama list

# Use in Python
pip install ollama
```

```python
import ollama

response = ollama.chat(model='llama3.2', messages=[
  {'role': 'user', 'content': 'Why is the sky blue?'}
])
print(response['message']['content'])
```

#### 2. **LM Studio** (GUI-based)
- Download from lmstudio.ai
- Browse and download models directly
- User-friendly interface
- Local API server included

#### 3. **llama.cpp** (Lightweight, CPU-friendly)
```bash
git clone https://github.com/ggerganov/llama.cpp
cd llama.cpp
make

# Download a GGUF model
# Run inference
./main -m models/llama-2-7b.gguf -p "Your prompt here"
```

### Option B: Running Traditional ML Models

#### Using scikit-learn
```python
import joblib
from sklearn.ensemble import RandomForestClassifier

# Train and save
model = RandomForestClassifier()
model.fit(X_train, y_train)
joblib.dump(model, 'model.pkl')

# Load and predict
loaded_model = joblib.load('model.pkl')
predictions = loaded_model.predict(X_test)
```

#### Using PyTorch
```python
import torch

# Load a model
model = torch.load('model.pth')
model.eval()

# Or load state dict
model = YourModelClass()
model.load_state_dict(torch.load('model_weights.pth'))
```

### Option C: Running Computer Vision Models

```python
# Using Hugging Face Transformers
from transformers import pipeline

# Image classification
classifier = pipeline("image-classification", 
                     model="google/vit-base-patch16-224")
results = classifier("image.jpg")

# Object detection
detector = pipeline("object-detection", 
                   model="facebook/detr-resnet-50")
```

---

## Part 2: Fine-tuning Models

### A. Fine-tuning LLMs

#### Using Hugging Face Transformers
```python
from transformers import AutoModelForCausalLM, AutoTokenizer, Trainer, TrainingArguments
from datasets import load_dataset

# Load model and tokenizer
model_name = "microsoft/phi-2"
model = AutoModelForCausalLM.from_pretrained(model_name)
tokenizer = AutoTokenizer.from_pretrained(model_name)

# Prepare dataset
dataset = load_dataset("your_dataset")

def tokenize_function(examples):
    return tokenizer(examples["text"], padding="max_length", truncation=True)

tokenized_datasets = dataset.map(tokenize_function, batched=True)

# Training arguments
training_args = TrainingArguments(
    output_dir="./results",
    num_train_epochs=3,
    per_device_train_batch_size=4,
    save_steps=1000,
    save_total_limit=2,
    learning_rate=2e-5
)

# Trainer
trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=tokenized_datasets["train"],
    eval_dataset=tokenized_datasets["test"]
)

trainer.train()
```

#### Using LoRA (Low-Rank Adaptation) - Memory Efficient
```python
from peft import LoraConfig, get_peft_model, prepare_model_for_kbit_training
from transformers import AutoModelForCausalLM

model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-2-7b-hf",
    load_in_8bit=True,
    device_map="auto"
)

model = prepare_model_for_kbit_training(model)

lora_config = LoraConfig(
    r=8,
    lora_alpha=32,
    target_modules=["q_proj", "v_proj"],
    lora_dropout=0.05,
    bias="none",
    task_type="CAUSAL_LM"
)

model = get_peft_model(model, lora_config)
# Continue with training...
```

#### Using Ollama's Modelfile (Simplest)
```dockerfile
# Create a Modelfile
FROM llama3.2

PARAMETER temperature 0.7
PARAMETER top_p 0.9

SYSTEM """
You are a helpful customer service assistant for TechCorp.
Always be polite and professional.
"""
```

```bash
# Create custom model
ollama create my-custom-model -f Modelfile
ollama run my-custom-model
```

### B. Fine-tuning Traditional ML Models

```python
# Hyperparameter tuning with GridSearchCV
from sklearn.model_selection import GridSearchCV
from sklearn.ensemble import RandomForestClassifier

param_grid = {
    'n_estimators': [100, 200, 300],
    'max_depth': [10, 20, 30],
    'min_samples_split': [2, 5, 10]
}

rf = RandomForestClassifier()
grid_search = GridSearchCV(rf, param_grid, cv=5)
grid_search.fit(X_train, y_train)

best_model = grid_search.best_estimator_
```

---

## Part 3: Deployment Options

### A. Local API Deployment

#### 1. FastAPI (Recommended)
```python
from fastapi import FastAPI
from pydantic import BaseModel
import torch

app = FastAPI()

# Load model once at startup
model = torch.load('model.pth')
model.eval()

class PredictionRequest(BaseModel):
    data: list

@app.post("/predict")
async def predict(request: PredictionRequest):
    with torch.no_grad():
        input_tensor = torch.tensor(request.data)
        prediction = model(input_tensor)
    return {"prediction": prediction.tolist()}

# Run with: uvicorn main:app --reload
```

#### 2. Flask (Simpler)
```python
from flask import Flask, request, jsonify
import joblib

app = Flask(__name__)
model = joblib.load('model.pkl')

@app.route('/predict', methods=['POST'])
def predict():
    data = request.json
    prediction = model.predict([data['features']])
    return jsonify({'prediction': prediction.tolist()})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### B. Docker Deployment

```dockerfile
# Dockerfile
FROM python:3.10-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```bash
# Build and run
docker build -t ml-model-api .
docker run -p 8000:8000 ml-model-api
```

### C. Cloud Deployment Options

#### 1. Hugging Face Spaces
```python
# app.py for Gradio interface
import gradio as gr
from transformers import pipeline

classifier = pipeline("sentiment-analysis")

def predict(text):
    result = classifier(text)[0]
    return f"{result['label']}: {result['score']:.2f}"

demo = gr.Interface(
    fn=predict,
    inputs="text",
    outputs="text"
)

demo.launch()
```

#### 2. AWS Lambda (Serverless)
```python
import json
import boto3
import pickle

def lambda_handler(event, context):
    # Load model from S3
    s3 = boto3.client('s3')
    response = s3.get_object(Bucket='my-bucket', Key='model.pkl')
    model = pickle.loads(response['Body'].read())
    
    # Predict
    data = json.loads(event['body'])
    prediction = model.predict([data['features']])
    
    return {
        'statusCode': 200,
        'body': json.dumps({'prediction': prediction.tolist()})
    }
```

### D. Production-Ready Deployment (BentoML)

```python
import bentoml
from bentoml.io import JSON

# Save model
bentoml.pytorch.save_model("my_model", model)

# Create service
@bentoml.service
class MyModelService:
    model = bentoml.pytorch.get("my_model:latest")
    
    @bentoml.api
    def predict(self, input_data: JSON) -> JSON:
        return self.model.predict(input_data)
```

```bash
# Build and serve
bentoml build
bentoml serve service:latest
bentoml containerize my_model:latest
```

---

## Quick Start Recommendations

### For Beginners:
1. **Running models**: Start with Ollama
2. **Fine-tuning**: Use Hugging Face AutoTrain
3. **Deployment**: FastAPI + Docker

### For Production:
1. **Running**: llama.cpp or vLLM for LLMs
2. **Fine-tuning**: LoRA/QLoRA for efficiency
3. **Deployment**: Kubernetes + BentoML or Ray Serve

### Hardware Requirements:
- **Small models (7B)**: 16GB RAM, GPU optional
- **Medium models (13B)**: 32GB RAM, 8GB+ GPU
- **Large models (70B)**: 64GB+ RAM, 24GB+ GPU or quantized versions

### Useful Tools:
- **Model repositories**: Hugging Face Hub, Ollama Library
- **Monitoring**: Weights & Biases, MLflow
- **Optimization**: ONNX Runtime, TensorRT
- **Quantization**: bitsandbytes, GPTQ, GGUF