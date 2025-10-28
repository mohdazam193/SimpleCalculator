# SimpleCalculator

# Simple Calculator App (Static HTML + JS)

This project contains a small **calculator** web app (HTML + JavaScript) packaged in a Docker image (nginx) and Kubernetes manifests to run it on a cluster.

## File tree

```
simple-calculator-k8s/
├── Dockerfile
├── index.html
├── README.md
└── k8s/
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
```

---

## index.html

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Simple Calculator</title>
  <style>
    body{font-family:system-ui,Segoe UI,Roboto,Arial;margin:2rem;background:#f6f8fa}
    .card{max-width:420px;margin:1rem auto;padding:1.25rem;background:white;border-radius:12px;box-shadow:0 6px 18px rgba(0,0,0,0.08)}
    .display{width:100%;height:60px;border-radius:8px;padding:0.5rem;font-size:1.5rem;border:1px solid #e6e9ee;text-align:right}
    .grid{display:grid;grid-template-columns:repeat(4,1fr);gap:8px;margin-top:12px}
    button{padding:14px;border-radius:8px;border:0;font-weight:600;cursor:pointer}
    .op{background:#f0f3f7}
    .equals{grid-column:span 2;background:#2563eb;color:white}
    .clear{background:#ef4444;color:white}
  </style>
</head>
<body>
  <div class="card">
    <h2>Calculator</h2>
    <input id="display" class="display" readonly value="0" />

    <div class="grid">
      <button class="clear" id="clear">C</button>
      <button class="op" data-op="%">%</button>
      <button class="op" data-op="/">÷</button>
      <button class="op" data-op="*">×</button>

      <button data-num>7</button>
      <button data-num>8</button>
      <button data-num>9</button>
      <button class="op" data-op="-">−</button>

      <button data-num>4</button>
      <button data-num>5</button>
      <button data-num>6</button>
      <button class="op" data-op="+">+</button>

      <button data-num>1</button>
      <button data-num>2</button>
      <button data-num>3</button>
      <button class="equals" id="equals">=</button>

      <button data-num>0</button>
      <button data-num>.</button>
      <button id="back">⌫</button>
    </div>
  </div>

  <script>
    const display = document.getElementById('display');
    let current = '';

    function updateDisplay(){
      display.value = current === '' ? '0' : current;
    }

    document.querySelectorAll('[data-num]').forEach(b => b.addEventListener('click', ()=>{
      const v = b.textContent;
      if(v === '.' && current.slice(-1) === '.') return;
      current += v;
      updateDisplay();
    }));

    document.querySelectorAll('[data-op]').forEach(b => b.addEventListener('click', ()=>{
      const op = b.getAttribute('data-op');
      if(current === '') return;
      // prevent two operators in a row
      if(/[+\-*/%]$/.test(current)) current = current.slice(0,-1);
      current += op;
      updateDisplay();
    }));

    document.getElementById('clear').addEventListener('click', ()=>{ current = ''; updateDisplay(); });
    document.getElementById('back').addEventListener('click', ()=>{ current = current.slice(0,-1); updateDisplay(); });

    document.getElementById('equals').addEventListener('click', ()=>{
      if(current === '') return;
      try{
        // replace ÷,× with JS equivalents
        const expr = current.replace(/×/g, '*').replace(/÷/g, '/').replace(/%/g, '%');
        // evaluate safely: allow only digits, operators, dot and %
        if(!/^[0-9+\-*/.()%\s]+$/.test(expr)) throw new Error('Invalid input');
        // handle percent operator: convert 50% -> (50/100)
        const handled = expr.replace(/(\d+(?:\.\d+)?)%/g, '($1/100)');
        const result = Function('return ' + handled)();
        current = String(result);
        updateDisplay();
      } catch (e){
        display.value = 'Error';
        current = '';
      }
    });

    // keyboard support
    window.addEventListener('keydown', (e)=>{
      if((/\d|\./).test(e.key)){
        current += e.key; updateDisplay();
      } else if(['+','-','/','*','%'].includes(e.key)){
        if(current === '') return; if(/[+\-*/%]$/.test(current)) current = current.slice(0,-1); current += e.key; updateDisplay();
      } else if(e.key === 'Enter') { document.getElementById('equals').click(); }
      else if(e.key === 'Backspace') { current = current.slice(0,-1); updateDisplay(); }
      else if(e.key === 'Escape') { current = ''; updateDisplay(); }
    });
  </script>
</body>
</html>
```

---

## Dockerfile

```Dockerfile
# Use nginx to serve the static HTML
FROM nginx:alpine
LABEL maintainer="you@example.com"

# Remove default nginx content and copy our app
RUN rm -rf /usr/share/nginx/html/*
COPY index.html /usr/share/nginx/html/index.html

# Expose port 80
EXPOSE 80

# default command from nginx image will run
```

---

## k8s/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: calculator-deploy
  labels:
    app: calculator
spec:
  replicas: 2
  selector:
    matchLabels:
      app: calculator
  template:
    metadata:
      labels:
        app: calculator
    spec:
      containers:
      - name: calculator
        image: <YOUR_REGISTRY>/simple-calculator:latest
        ports:
        - containerPort: 80
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 2
          periodSeconds: 5
```

---

## k8s/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: calculator-svc
spec:
  selector:
    app: calculator
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http
  type: ClusterIP
```

---

## k8s/ingress.yaml (optional)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: calculator-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: calculator.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: calculator-svc
            port:
              number: 80
```

---

## README.md

```markdown
# Simple Calculator (static)

A tiny static calculator app that can be containerized and deployed to Kubernetes.

## Build & run locally with Docker

1. Build image:

```bash
docker build -t simple-calculator:latest .
```

2. Run:

```bash
docker run --rm -p 8080:80 simple-calculator:latest
# then open http://localhost:8080
```

## Push to registry (example)

```bash
docker tag simple-calculator:latest <YOUR_REGISTRY>/simple-calculator:latest
docker push <YOUR_REGISTRY>/simple-calculator:latest
```

## Deploy to Kubernetes

Replace `<YOUR_REGISTRY>/simple-calculator:latest` in `k8s/deployment.yaml` with your image ref.

```bash
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
# optional ingress
kubectl apply -f k8s/ingress.yaml
```

## Notes
- The app is intentionally tiny and uses no backend.
- For production, push the image to a registry and use secure ingress configuration.
