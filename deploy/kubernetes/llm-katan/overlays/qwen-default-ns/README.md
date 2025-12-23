# llm-katan Qwen Default Namespace Overlay

## Overview

This Kustomize overlay deploys llm-katan with Qwen3-0.6B to the `default` namespace for AI Gateway integration.

## Why Default Namespace?

The AI Gateway resources (`Gateway`, `AIGatewayRoute`, `Backend`) are deployed to the `default` namespace. Kubernetes services can only reference backends in the same namespace, so llm-katan **must** be deployed to `default` for proper routing.

## What This Overlay Does

1. **Namespace**: Changes deployment from `llm-katan` namespace to `default`
2. **Service Name**: Adds `-qwen` suffix → `llm-katan-qwen`
3. **Labels**: Adds identification labels for tracking

## Important Notes

### No LoRA Support

llm-katan (Qwen3-0.6B) **does not support LoRA adapters**. This overlay uses the base Qwen3-0.6B model for all requests.

**What this means:**

- ✅ All routing infrastructure works correctly
- ✅ Requests are delivered to llm-katan
- ✅ Real LLM responses are generated
- ❌ No domain-specific specialization (math, science, law, etc.)
- ❌ All "expert" models behave identically

### Use Cases

**Recommended for:**

- Infrastructure testing (routing, networking, deployment)
- API compatibility validation
- Performance and load testing
- Integration testing

**Not recommended for:**

- LoRA adapter validation
- Domain-specific expertise testing
- Specialized response quality testing

## Deployment

```bash
kubectl apply -k deploy/kubernetes/llm-katan/overlays/qwen-default-ns
```

## Resources Created

- **Deployment**: `llm-katan-qwen` (in `default` namespace)
- **Service**: `llm-katan-qwen` (port 8000)
- **PVC**: `llm-katan-models-qwen` (for model storage)

## Model Details

- **Model**: Qwen3-0.6B
- **Backend**: transformers
- **Device**: CPU
- **Quantization**: int8 (enabled)
- **API**: OpenAI-compatible

## Service Endpoint

```
http://llm-katan-qwen.default.svc.cluster.local:8000
```

## Health Check

```bash
kubectl exec -it deployment/llm-katan-qwen -- curl http://localhost:8000/health
```

Expected response:

```json
{"status": "healthy"}
```

## Alternative: vLLM Simulator with LoRA

If you need LoRA adapter testing, use the original vLLM simulator:

```bash
kubectl apply -f deploy/kubernetes/ai-gateway/aigw-resources/base-model.yaml
```

This deploys `ghcr.io/llm-d/llm-d-inference-sim:v0.5.0` which simulates LoRA adapter behavior.
