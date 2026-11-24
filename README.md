agentic-rag-system

A RAG system where an LLM agent decides where to look for an answer (PDFs, websites or its own knowledge) before it replies. It runs local models with Ollama (gemma3:270m), stores vectors in Oracle AI Database, ingests PDFs with Docling and supports agent-to-agent calls (A2A). The UI is Gradio.

The application is Oracle's, from oracle-devrel/ai-solutions (apps/agentic_rag). I did not write the agent code. What I did was take it to Kubernetes and check how it behaves there.


What I did

  Built the image and deployed it to a local minikube cluster (docker driver) with the manifests in k8s/local-deployment.
  Got the pod to Running, with Ollama and the Gradio app up inside the same container.
  Checked scheduling, resource requests and limits, volumes and config, then tore everything down: manifests, cluster and images.


Run it on minikube

  minikube start --driver=docker --cpus=4 --memory=12g
  docker build --network=host -t agentic-rag:k8s-test-v2 .
  minikube image load agentic-rag:k8s-test-v2
  kubectl apply -f k8s/local-deployment/
  kubectl get pods -w
  kubectl logs -f -l app=agentic-rag
  kubectl port-forward svc/agentic-rag 7860:80

  Then open http://localhost:7860. The logs should show "Ollama is ready." and "Starting Gradio application...".


Clean up

  kubectl delete -f k8s/local-deployment/
  minikube delete
  docker rmi agentic-rag:k8s-test-v2


What I would change before running this anywhere real

  The pod asks for 8Gi of memory and 2 CPUs just to be scheduled, and up to 100Gi of ephemeral storage. That fits a workstation, not a small node pool.

  Ollama and the app share one container. If Ollama dies, the whole pod restarts, and they can't be scaled separately. They should be two Deployments.

  The image is heavy: it bakes in the model, build tools and the NVIDIA toolkit repo. A multi-stage build would cut it down a lot.

  The Hugging Face token lives in a ConfigMap. It belongs in a Secret.

  pvcs.yaml creates ollama-models-pvc, but nothing mounts it. It only takes up 50Gi.

  The Service is type LoadBalancer, which minikube can't provide without minikube tunnel. port-forward is simpler for local work.

  There are no liveness or readiness probes, so Kubernetes marks the pod ready before Ollama finishes loading.
