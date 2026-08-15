# vLLM AI Interview Inference Server

A scalable LLM inference server for an AI Interview System using **vLLM** and an open-source Hugging Face model.

The service exposes an **OpenAI-compatible API** that can be consumed by a Node.js/Express backend.

---

## Architecture

```text
                         ┌──────────────────┐
                         │   React Frontend │
                         └────────┬─────────┘
                                  │
                                  ▼
                         ┌──────────────────┐
                         │ Node.js Backend  │
                         │    Express API   │
                         └────────┬─────────┘
                                  │
                         HTTP / OpenAI API
                                  │
                                  ▼
                    ┌──────────────────────────┐
                    │       vLLM Server        │
                    │                          │
                    │ Request Scheduler        │
                    │ KV Cache                  │
                    │ Continuous Batching      │
                    └────────────┬─────────────┘
                                 │
                                 ▼
                            ┌──────────┐
                            │   GPU    │
                            │          │
                            │ LLM      │
                            └──────────┘

             ┌─────────────────────────────┐
             │        MongoDB              │
             │                             │
             │ Questions                   │
             │ Expected Answers            │
             │ Evaluation Criteria         │
             │ Interview Results           │
             └─────────────────────────────┘
```

---

# 1. What is vLLM?

vLLM is an open-source **LLM inference and serving engine**.

It does not train the model.

Instead:

```text
Hugging Face Model
        ↓
      vLLM
        ↓
      GPU
        ↓
OpenAI-compatible API
        ↓
Node.js Backend
```

vLLM keeps the model loaded in GPU memory and efficiently handles multiple inference requests.

---

# 2. Requirements

## Hardware

A CUDA-compatible NVIDIA GPU is recommended.

For a small 7B/8B model, a GPU with approximately **12–16 GB VRAM** can be a starting point, although the actual requirement depends on:

- Model size
- Quantization
- Context length
- Number of concurrent requests
- KV cache usage
- Maximum output length

For production, benchmark the exact model and workload before choosing hardware.

## Software

Recommended:

```text
Ubuntu 22.04+
Python 3.10+
NVIDIA Driver
CUDA-compatible environment
Docker
Git
```

Check the GPU:

```bash
nvidia-smi
```

---

# 3. Choose a Model

vLLM normally loads models from Hugging Face or a local model directory.

Example:

```text
Qwen/Qwen3-8B
```

Other model families that may be suitable include:

```text
Qwen
Llama
Gemma
Mistral
```

Choose a model based on:

- Quality
- VRAM requirements
- Context length
- Inference speed
- License
- Concurrent-user requirements

Do not assume that a larger model is automatically better for an interview system.

---

# 4. Install vLLM

Create a virtual environment:

```bash
python3 -m venv vllm-env
```

Activate it:

```bash
source vllm-env/bin/activate
```

Upgrade pip:

```bash
pip install --upgrade pip
```

Install vLLM:

```bash
pip install vllm
```

Verify:

```bash
vllm --help
```

---

# 5. Start vLLM

Example:

```bash
vllm serve Qwen/Qwen3-8B
```

By default, the API server runs on:

```text
http://localhost:8000
```

The API is OpenAI-compatible.

---

# 6. Configure the Server

A basic production-oriented command can look like:

```bash
vllm serve Qwen/Qwen3-8B \
    --host 0.0.0.0 \
    --port 8000 \
    --gpu-memory-utilization 0.90
```

The exact parameters should be tuned according to your GPU and workload.

Do not blindly copy GPU-memory settings from another machine.

---

# 7. Test the Server

Once vLLM is running, check the models endpoint:

```bash
curl http://localhost:8000/v1/models
```

You should receive information about the loaded model.

---

# 8. Test Chat Completion

Example:

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-8B",
    "messages": [
      {
        "role": "system",
        "content": "You are an AI technical interviewer."
      },
      {
        "role": "user",
        "content": "Explain polymorphism in Java."
      }
    ],
    "temperature": 0.2,
    "max_tokens": 300
  }'
```

The response will contain the generated answer.

---

# 9. Connecting Node.js Backend

Your Node.js application should communicate with vLLM.

```text
React
  ↓
Node.js
  ↓
vLLM API
  ↓
LLM
```

Example using the OpenAI SDK:

```bash
npm install openai
```

Example:

```javascript
import OpenAI from "openai";

const client = new OpenAI({
    baseURL: "http://localhost:8000/v1",
    apiKey: "dummy"
});

const response = await client.chat.completions.create({
    model: "Qwen/Qwen3-8B",
    messages: [
        {
            role: "system",
            content: "You are an AI technical interviewer."
        },
        {
            role: "user",
            content: "Evaluate this candidate's answer."
        }
    ],
    temperature: 0.2
});

console.log(response.choices[0].message.content);
```

The API key value may not be used for authentication unless you configure authentication on the server. Do not expose real server credentials in the frontend.

---

# 10. AI Interview Architecture

The LLM should not be responsible for storing your interview questions.

Use your database for that.

```text
MongoDB

questions
├── question
├── category
├── difficulty
├── expectedAnswer
├── keyConcepts
├── evaluationCriteria
└── followUpQuestions
```

Example:

```json
{
    "question": "What is polymorphism in Java?",
    "category": "Java",
    "difficulty": "medium",
    "keyConcepts": [
        "method overloading",
        "method overriding",
        "runtime polymorphism"
    ],
    "evaluationCriteria": [
        "Defines polymorphism",
        "Mentions multiple forms",
        "Understands overriding"
    ]
}
```

---

# 11. Database + vLLM Flow

When a candidate answers:

```text
Candidate Answer
       ↓
Node.js Backend
       ↓
MongoDB
       ↓
Retrieve Question
       ↓
Retrieve Evaluation Criteria
       ↓
Build AI Prompt
       ↓
vLLM
       ↓
LLM
       ↓
Evaluation
       ↓
Node.js
       ↓
MongoDB
       ↓
Frontend
```

Example prompt:

```text
SYSTEM:
You are an AI technical interviewer.

You must evaluate the candidate using the supplied
evaluation criteria.

QUESTION:
What is polymorphism in Java?

KEY CONCEPTS:
- Method overloading
- Method overriding
- Runtime polymorphism

EVALUATION CRITERIA:
- Defines polymorphism
- Explains multiple forms
- Understands overriding

CANDIDATE ANSWER:
[Candidate's answer]

Return:

{
    "score": 0-10,
    "correct": true/false,
    "strengths": [],
    "weaknesses": [],
    "feedback": "",
    "followUpQuestion": ""
}
```

---

# 12. Database-Grounded Responses

If the system should primarily rely on your stored interview content, use a **database-grounded prompting/RAG architecture**.

```text
User
 ↓
Backend
 ↓
Retrieve relevant data
 ↓
Build context
 ↓
vLLM
 ↓
LLM
 ↓
Validate response
 ↓
Return result
```

Important:

Prompting the model to use only supplied information is **not a mathematical guarantee** that it will never use pretrained knowledge.

For stricter control, combine:

```text
Database retrieval
+
Structured prompts
+
JSON schema/output validation
+
Application-level validation
```

For fixed questions, normal MongoDB lookup may be enough.

You do not necessarily need a vector database.

---

# 13. Store Results in MongoDB

vLLM should not be your permanent storage system.

Store results in your backend database:

```json
{
    "userId": "123",
    "questionId": "456",
    "answer": "Candidate answer...",
    "score": 8,
    "feedback": "Good understanding...",
    "timestamp": "2026-08-15T10:00:00Z"
}
```

The separation should be:

```text
vLLM
→ Inference

MongoDB
→ Permanent application data
```

---

# 14. Concurrency

For multiple users:

```text
User 1 ─┐
User 2 ─┤
User 3 ─┤
User 4 ─┤
         ├──→ Node.js
User N ─┘
             ↓
          Redis
             ↓
         AI Workers
             ↓
           vLLM
             ↓
            GPU
```

vLLM performs request scheduling and batching internally.

For larger systems, Redis can be used for:

- Job queues
- Rate limiting
- Session state
- Background AI tasks
- Retry handling

Do not create a new vLLM process for every user.

Run a persistent vLLM server and allow it to manage requests.

---

# 15. Production Architecture

```text
                    INTERNET
                       │
                       ▼
                ┌──────────────┐
                │    Vercel    │
                │   Frontend   │
                └──────┬───────┘
                       │
                       ▼
                ┌──────────────┐
                │ Node.js API  │
                └──────┬───────┘
                       │
             ┌─────────┼─────────┐
             ▼         ▼         ▼
          MongoDB    Redis      vLLM
           Atlas                Server
                                  │
                                  ▼
                                GPU
                                  │
                                  ▼
                                LLM
```

For a 50–100-user target, start with a single well-sized GPU server and benchmark it before adding multiple inference servers.

---

# 16. Security

Do not expose an unrestricted vLLM endpoint directly to the internet.

Avoid:

```text
Internet → :8000 → vLLM
```

Prefer:

```text
Internet
   ↓
HTTPS
   ↓
Reverse Proxy
   ↓
Authentication / Rate Limiting
   ↓
vLLM
```

Your frontend should communicate with your Node.js backend rather than directly with the vLLM server.

```text
Correct:

React → Node.js → vLLM

Avoid:

React → vLLM
```

This keeps your AI infrastructure private and allows the backend to enforce:

- Authentication
- Authorization
- Rate limits
- Prompt restrictions
- Request validation
- Usage limits

---

# 17. Environment Variables

Example backend `.env`:

```env
VLLM_BASE_URL=http://localhost:8000/v1
VLLM_MODEL=Qwen/Qwen3-8B

MONGODB_URI=your_mongodb_connection
REDIS_URL=your_redis_connection
```

Never commit `.env` files:

```gitignore
.env
.env.*
```

---

# 18. Docker Deployment

For a production GPU server, Docker is recommended.

Example:

```bash
docker run --runtime nvidia --gpus all \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  -p 8000:8000 \
  --ipc=host \
  vllm/vllm-openai:latest \
  --model Qwen/Qwen3-8B
```

The exact Docker image tag and command should be matched to the installed NVIDIA/CUDA environment and current vLLM documentation.

---

# 19. Monitoring

Track:

```text
GPU utilization
GPU VRAM
Request latency
Tokens/second
Requests/second
Active requests
Queue length
Error rate
KV-cache usage
```

Useful commands:

```bash
nvidia-smi
```

For load testing, simulate realistic interview traffic instead of simply counting registered users.

Important metrics:

```text
Concurrent users
        ↓
Concurrent LLM requests
        ↓
Tokens per request
        ↓
GPU utilization
        ↓
Response latency
```

---

# 20. Scaling

Start:

```text
Node.js
   ↓
Redis
   ↓
vLLM
   ↓
1 GPU
```

If the GPU becomes the bottleneck:

```text
                 Load Balancer
                  /          \
                 ↓            ↓
              vLLM #1      vLLM #2
                 │            │
                GPU          GPU
```

The Node.js backend can then distribute inference requests between inference servers.

---

# 21. Important Concepts

### LLM

The actual trained model:

```text
Qwen
Llama
Gemma
Mistral
```

### vLLM

The inference/serving engine:

```text
Model
 ↓
vLLM
 ↓
GPU
 ↓
API
```

### MongoDB

Stores:

```text
Questions
Answers
Users
Scores
Feedback
Interview history
```

### Redis

Optional infrastructure for:

```text
Queues
Caching
Rate limiting
Background jobs
```

### Node.js

Controls:

```text
Authentication
Interview logic
Question selection
Database access
AI requests
Validation
API responses
```

---

# 22. Final System

```text
                         USER
                           │
                           ▼
                    React Frontend
                           │
                           ▼
                    Node.js Backend
                           │
             ┌─────────────┼─────────────┐
             │             │             │
             ▼             ▼             ▼
          MongoDB        Redis          vLLM
             │             │             │
             │             │             ▼
             │             │            GPU
             │             │             │
             │             │             ▼
             │             │            LLM
             │             │             │
             │             └─────────────┤
             │                           │
             └────────── Results ─────────┘
```

## Goal

The final system should have this responsibility split:

```text
Frontend
→ User interface

Node.js
→ Business logic

MongoDB
→ Interview/question/result storage

Redis
→ Queue/cache/scaling

vLLM
→ LLM inference

GPU
→ Model computation

LLM
→ Question generation/evaluation/feedback
```

---

## Recommended Development Path

### Phase 1

Run everything locally:

```text
React
Node.js
MongoDB
Ollama/vLLM
```

### Phase 2

Move the LLM to a dedicated GPU server:

```text
React → Node.js → vLLM → GPU
```

### Phase 3

Add Redis:

```text
Node.js → Redis → vLLM
```

### Phase 4

Load test with simulated concurrent interview users.

### Phase 5

If required, scale horizontally:

```text
             Load Balancer
              /          \
             ▼            ▼
          vLLM #1      vLLM #2
            GPU           GPU
```

This gives you a clean path from a local college project to a system capable of supporting significantly more concurrent users.