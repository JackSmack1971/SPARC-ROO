        app.get("/metrics")(self.get_metrics)
        app.post("/reload-model")(self.reload_model)
        
        return app
    
    def _load_model(self) -> None:
        """Load model with error handling and validation"""
        try:
            logger.info(f"Loading model from {self.model_path}")
            
            # Determine model class and load
            metadata_path = self.model_path / "metadata.json"
            if not metadata_path.exists():
                raise FileNotFoundError(f"Model metadata not found at {metadata_path}")
            
            import json
            with open(metadata_path, 'r') as f:
                metadata = json.load(f)
            
            # Dynamically import and instantiate model
            model_class_name = metadata.get('model_type', 'SPARCClassificationModel')
            self.model = self._instantiate_model(model_class_name)
            self.model.load_model(self.model_path)
            
            self.model_version = metadata.get('version', 'unknown')
            
            logger.info(f"Model loaded successfully. Version: {self.model_version}")
            
        except Exception as e:
            logger.error(f"Failed to load model: {str(e)}")
            raise
    
    def _instantiate_model(self, class_name: str) -> SPARCBaseModel:
        """Dynamically instantiate model class"""
        from ..models.training import SPARCClassificationModel
        
        class_map = {
            'SPARCClassificationModel': SPARCClassificationModel,
            # Add other model types as needed
        }
        
        if class_name not in class_map:
            raise ValueError(f"Unknown model class: {class_name}")
        
        return class_map[class_name](self.config)
    
    async def predict(self, request: PredictionRequest, 
                     background_tasks: BackgroundTasks) -> PredictionResponse:
        """Handle prediction requests with comprehensive monitoring"""
        start_time = time.time()
        prediction_id = str(uuid.uuid4())
        
        try:
            logger.info(f"Processing prediction request {prediction_id} with {len(request.instances)} instances")
            
            if not self.model:
                raise HTTPException(status_code=503, detail="Model not loaded")
            
            # Convert instances to DataFrame
            df = pd.DataFrame(request.instances)
            
            # Validate input data
            validation_result = self.data_validator.validate_dataframe(df)
            if not validation_result['is_valid']:
                raise HTTPException(
                    status_code=400, 
                    detail=f"Invalid input data: {validation_result['errors']}"
                )
            
            # Generate predictions
            predictions = self.model.predict(df).tolist()
            
            # Generate probabilities if requested
            probabilities = None
            if request.include_probabilities:
                try:
                    probabilities = self.model.predict_proba(df).tolist()
                except AttributeError:
                    logger.warning("Model does not support probability predictions")
            
            processing_time = (time.time() - start_time) * 1000
            
            # Create response
            response = PredictionResponse(
                predictions=predictions,
                probabilities=probabilities,
                model_version=self.model_version,
                prediction_id=prediction_id,
                timestamp=datetime.utcnow().isoformat(),
                processing_time_ms=processing_time
            )
            
            # Log metrics asynchronously
            background_tasks.add_task(
                self._log_prediction_metrics,
                prediction_id, len(request.instances), processing_time, True
            )
            
            logger.info(f"Prediction {prediction_id} completed in {processing_time:.2f}ms")
            
            return response
            
        except HTTPException:
            # Re-raise HTTP exceptions
            background_tasks.add_task(
                self._log_prediction_metrics,
                prediction_id, len(request.instances), 
                (time.time() - start_time) * 1000, False
            )
            raise
        except Exception as e:
            logger.error(f"Prediction error for {prediction_id}: {str(e)}")
            background_tasks.add_task(
                self._log_prediction_metrics,
                prediction_id, len(request.instances), 
                (time.time() - start_time) * 1000, False
            )
            raise HTTPException(status_code=500, detail=f"Prediction failed: {str(e)}")
    
    async def health_check(self) -> HealthResponse:
        """Health check endpoint"""
        return HealthResponse(
            status="healthy" if self.model else "unhealthy",
            model_loaded=self.model is not None,
            model_version=self.model_version,
            timestamp=datetime.utcnow().isoformat(),
            uptime_seconds=time.time() - self.start_time
        )
    
    async def get_metrics(self) -> Dict[str, Any]:
        """Get current server metrics"""
        return {
            "uptime_seconds": time.time() - self.start_time,
            "model_version": self.model_version,
            "prediction_metrics": self.metrics_collector.get_summary(),
            "system_metrics": self._get_system_metrics()
        }
    
    async def reload_model(self) -> Dict[str, str]:
        """Reload model from disk"""
        try:
            self._load_model()
            return {"status": "success", "message": f"Model reloaded. Version: {self.model_version}"}
        except Exception as e:
            logger.error(f"Model reload failed: {str(e)}")
            raise HTTPException(status_code=500, detail=f"Model reload failed: {str(e)}")
    
    async def _log_prediction_metrics(self, prediction_id: str, batch_size: int, 
                                    processing_time: float, success: bool) -> None:
        """Log prediction metrics asynchronously"""
        self.metrics_collector.log_prediction_request(
            prediction_id=prediction_id,
            batch_size=batch_size,
            processing_time_ms=processing_time,
            success=success,
            model_version=self.model_version
        )
    
    def _get_system_metrics(self) -> Dict[str, Any]:
        """Get system performance metrics"""
        import psutil
        
        return {
            "cpu_percent": psutil.cpu_percent(),
            "memory_percent": psutil.virtual_memory().percent,
            "disk_percent": psutil.disk_usage('/').percent,
            "load_average": psutil.getloadavg() if hasattr(psutil, 'getloadavg') else None
        }

# Application factory
def create_app(model_path: str, config_path: str) -> FastAPI:
    """Create FastAPI application with loaded model"""
    import yaml
    
    # Load configuration
    with open(config_path, 'r') as f:
        config = yaml.safe_load(f)
    
    # Create server instance
    server = SPARCModelServer(Path(model_path), config)
    
    return server.app

if __name__ == "__main__":
    import uvicorn
    import argparse
    
    parser = argparse.ArgumentParser(description="SPARC ML Model Server")
    parser.add_argument("--model-path", required=True, help="Path to model directory")
    parser.add_argument("--config-path", required=True, help="Path to configuration file")
    parser.add_argument("--host", default="0.0.0.0", help="Host address")
    parser.add_argument("--port", type=int, default=8000, help="Port number")
    parser.add_argument("--workers", type=int, default=1, help="Number of workers")
    
    args = parser.parse_args()
    
    app = create_app(args.model_path, args.config_path)
    
    uvicorn.run(
        app,
        host=args.host,
        port=args.port,
        workers=args.workers,
        log_level="info"
    )
```

### Infrastructure and Deployment

#### Kubernetes Deployment Configuration

**Model Serving Deployment**:
```yaml
# infrastructure/kubernetes/model-serving/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sparc-ml-model-server
  namespace: ml-production
  labels:
    app: sparc-ml-model
    component: model-server
    version: v1.0.0
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: sparc-ml-model
      component: model-server
  template:
    metadata:
      labels:
        app: sparc-ml-model
        component: model-server
        version: v1.0.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8000"
        prometheus.io/path: "/metrics"
    spec:
      containers:
      - name: model-server
        image: your-registry/sparc-ml-model:latest
        ports:
        - containerPort: 8000
          name: http
        env:
        - name: MODEL_PATH
          value: "/app/models/current"
        - name: CONFIG_PATH
          value: "/app/config/prod_config.yaml"
        - name: LOG_LEVEL
          value: "INFO"
        resources:
          requests:
            cpu: 200m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 2Gi
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 2
        volumeMounts:
        - name: model-storage
          mountPath: /app/models
          readOnly: true
        - name: config-volume
          mountPath: /app/config
          readOnly: true
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          readOnlyRootFilesystem: true
          allowPrivilegeEscalation: false
      volumes:
      - name: model-storage
        persistentVolumeClaim:
          claimName: ml-model-storage
      - name: config-volume
        configMap:
          name: ml-model-config
      nodeSelector:
        node-type: ml-inference
      tolerations:
      - key: ml-workload
        operator: Equal
        value: inference
        effect: NoSchedule

---
apiVersion: v1
kind: Service
metadata:
  name: sparc-ml-model-service
  namespace: ml-production
  labels:
    app: sparc-ml-model
    component: model-server
spec:
  type: ClusterIP
  selector:
    app: sparc-ml-model
    component: model-server
  ports:
  - name: http
    port: 80
    targetPort: 8000
    protocol: TCP

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sparc-ml-model-ingress
  namespace: ml-production
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - ml-api.your-domain.com
    secretName: ml-api-tls
  rules:
  - host: ml-api.your-domain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: sparc-ml-model-service
            port:
              number: 80
```

**Horizontal Pod Autoscaler**:
```yaml
# infrastructure/kubernetes/model-serving/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: sparc-ml-model-hpa
  namespace: ml-production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: sparc-ml-model-server
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
```

### Continuous Integration and Deployment

#### CI/CD Pipeline Configuration

**GitHub Actions Workflow**:
```yaml
# .github/workflows/ml-pipeline.yml
name: SPARC ML Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  PYTHON_VERSION: "3.11"
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}/ml-model

jobs:
  data-validation:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: ${{ env.PYTHON_VERSION }}
    
    - name: Install dependencies
      run: |
        pip install -r requirements/dev.txt
    
    - name: Run data validation tests
      run: |
        python -m pytest tests/unit/test_data_validation.py -v
        python -m pytest tests/integration/test_data_pipeline.py -v
    
    - name: Check data quality
      run: |
        python scripts/check_data_quality.py --config configs/dev_config.yaml

  model-testing:
    needs: data-validation
    runs-on: ubuntu-latest
    strategy:
      matrix:
        model-type: [random_forest, logistic_regression]
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: ${{ env.PYTHON_VERSION }}
    
    - name: Install dependencies
      run: |
        pip install -r requirements/training.txt
    
    - name: Run model tests
      run: |
        python -m pytest tests/model/ -v --model-type=${{ matrix.model-type }}
    
    - name: Train and validate model
      run: |
        python scripts/train_model.py \
          --config configs/ci_config.yaml \
          --model-type ${{ matrix.model-type }} \
          --experiment-name "ci-${{ github.sha }}-${{ matrix.model-type }}"
    
    - name: Upload model artifacts
      uses: actions/upload-artifact@v3
      with:
        name: model-${{ matrix.model-type }}-${{ github.sha }}
        path: models/
        retention-days: 30

  security-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Run security scan
      uses: securecodewarrior/github-action-security-scan@v1
      with:
        token: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Scan for secrets
      uses: trufflesecurity/trufflehog@v3.63.2
      with:
        path: ./
        base: main
        head: HEAD

  build-and-push:
    needs: [model-testing, security-scan]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    permissions:
      contents: read
      packages: write
    steps:
    - uses: actions/checkout@v4
    
    - name: Download model artifacts
      uses: actions/download-artifact@v3
      with:
        pattern: model-*
        path: models/
    
    - name: Log in to Container Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=sha,prefix={{branch}}-
          type=raw,value=latest,enable={{is_default_branch}}
    
    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        file: infrastructure/docker/serving.Dockerfile
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}

  deploy-staging:
    needs: build-and-push
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    environment: staging
    steps:
    - uses: actions/checkout@v4
    
    - name: Deploy to Staging
      run: |
        # Configure kubectl and deploy to staging
        echo "Deploying to staging environment"
        # Add deployment commands here

  deploy-production:
    needs: build-and-push
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
    - uses: actions/checkout@v4
    
    - name: Deploy to Production
      run: |
        # Configure kubectl and deploy to production
        echo "Deploying to production environment"
        # Add deployment commands here
```

### Monitoring and Observability

#### Comprehensive ML Monitoring

**Prometheus Monitoring Configuration**:
```yaml
# infrastructure/monitoring/prometheus-config.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "ml_alerting_rules.yml"

scrape_configs:
- job_name: 'sparc-ml-models'
  kubernetes_sd_configs:
  - role: pod
    namespaces:
      names:
      - ml-production
      - ml-staging
  relabel_configs:
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
    action: keep
    regex: true
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
    action: replace
    target_label: __metrics_path__
    regex: (.+)
  - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
    action: replace
    regex: ([^:]+)(?::\d+)?;(\d+)
    replacement: $1:$2
    target_label: __address__

alerting:
  alertmanagers:
  - kubernetes_sd_configs:
    - role: pod
      namespaces:
        names:
        - monitoring
```

**ML-Specific Alerting Rules**:
```yaml
# infrastructure/monitoring/ml_alerting_rules.yml
groups:
- name: ml-model-alerts
  rules:
  - alert: ModelPredictionLatencyHigh
    expr: histogram_quantile(0.95, rate(ml_prediction_duration_seconds_bucket[5m])) > 1.0
    for: 2m
    labels:
      severity: warning
      component: ml-model
    annotations:
      summary: "High prediction latency detected"
      description: "95th percentile latency is {{ $value }}s for model {{ $labels.model_name }}"
  
  - alert: ModelErrorRateHigh
    expr: rate(ml_prediction_errors_total[5m]) / rate(ml_predictions_total[5m]) > 0.05
    for: 1m
    labels:
      severity: critical
      component: ml-model
    annotations:
      summary: "High model error rate"
      description: "Error rate is {{ $value | humanizePercentage }} for model {{ $labels.model_name }}"
  
  - alert: ModelDriftDetected
    expr: ml_model_drift_score > 0.1
    for: 5m
    labels:
      severity: warning
      component: ml-model
    annotations:
      summary: "Model drift detected"
      description: "Drift score is {{ $value }} for model {{ $labels.model_name }}"
  
  - alert: FeatureDriftDetected
    expr: ml_feature_drift_score > 0.15
    for: 5m
    labels:
      severity: warning
      component: ml-model
    annotations:
      summary: "Feature drift detected"
      description: "Feature drift score is {{ $value }} for feature {{ $labels.feature_name }}"
  
  - alert: ModelAccuracyDegraded
    expr: ml_model_accuracy < 0.85
    for: 10m
    labels:
      severity: critical
      component: ml-model
    annotations:
      summary: "Model accuracy degraded"
      description: "Model accuracy is {{ $value | humanizePercentage }} for model {{ $labels.model_name }}"
```

### Quality Assurance and Testing

#### Comprehensive Testing Strategy

**Model Testing Framework**:
```python
# tests/model/test_model_performance.py
import pytest
import pandas as pd
import numpy as np
from sklearn.metrics import accuracy_score, precision_score, recall_score
from sklearn.model_selection import train_test_split

from src.models.training import SPARCClassificationModel
from src.data.validation import DataValidator

class TestModelPerformance:
    """
    SPARC Model Performance Tests
    
    Ensures model quality meets business requirements
    """
    
    @pytest.fixture
    def sample_data(self):
        """Generate sample data for testing"""
        np.random.seed(42)
        n_samples = 1000
        n_features = 10
        
        X = pd.DataFrame(
            np.random.randn(n_samples, n_features),
            columns=[f'feature_{i}' for i in range(n_features)]
        )
        
        # Create target with some signal
        y = pd.Series((X['feature_0'] + X['feature_1'] > 0).astype(int))
        
        return train_test_split(X, y, test_size=0.3, random_state=42)
    
    @pytest.fixture
    def trained_model(self, sample_data):
        """Train model for testing"""
        X_train, X_test, y_train, y_test = sample_data
        
        config = {
            'model_type': 'random_forest',
            'model': {
                'n_estimators': 10,  # Small for fast testing
                'random_state': 42
            },
            'cross_validation': {
                'n_folds': 3,
                'scoring': 'accuracy'
            }
        }
        
        model = SPARCClassificationModel(config)
        model.train(X_train, y_train, X_test, y_test)
        
        return model, X_test, y_test
    
    def test_model_accuracy_threshold(self, trained_model):
        """Test that model meets minimum accuracy threshold"""
        model, X_test, y_test = trained_model
        
        predictions = model.predict(X_test)
        accuracy = accuracy_score(y_test, predictions)
        
        # SPARC requirement: minimum 70% accuracy for acceptance
        assert accuracy >= 0.70, f"Model accuracy {accuracy:.3f} below threshold 0.70"
    
    def test_model_precision_recall(self, trained_model):
        """Test model precision and recall meet requirements"""
        model, X_test, y_test = trained_model
        
        predictions = model.predict(X_test)
        precision = precision_score(y_test, predictions, average='weighted')
        recall = recall_score(y_test, predictions, average='weighted')
        
        # SPARC requirements for balanced performance
        assert precision >= 0.65, f"Model precision {precision:.3f} below threshold 0.65"
        assert recall >= 0.65, f"Model recall {recall:.3f} below threshold 0.65"
    
    def test_model_prediction_consistency(self, trained_model):
        """Test that model predictions are consistent"""
        model, X_test, y_test = trained_model
        
        # Generate predictions multiple times
        predictions_1 = model.predict(X_test)
        predictions_2 = model.predict(X_test)
        
        # Predictions should be identical
        np.testing.assert_array_equal(predictions_1, predictions_2,
                                    "Model predictions are not consistent")
    
    def test_model_handles_edge_cases(self, trained_model):
        """Test model behavior with edge cases"""
        model, X_test, y_test = trained_model
        
        # Test with all zeros
        zero_data = pd.DataFrame(np.zeros((5, len(model.feature_names))),
                               columns=model.feature_names)
        predictions = model.predict(zero_data)
        assert len(predictions) == 5, "Model failed to handle zero input"
        
        # Test with extreme values
        extreme_data = pd.DataFrame(np.full((5, len(model.feature_names)), 1000),
                                  columns=model.feature_names)
        predictions = model.predict(extreme_data)
        assert len(predictions) == 5, "Model failed to handle extreme values"
    
    def test_model_feature_importance(self, trained_model):
        """Test that model provides feature importance"""
        model, X_test, y_test = trained_model
        
        importance = model.get_feature_importance()
        
        if importance is not None:
            assert len(importance) == len(model.feature_names), \
                "Feature importance length mismatch"
            assert all(imp >= 0 for imp in importance.values()), \
                "Feature importance contains negative values"
    
    def test_model_serialization(self, trained_model, tmp_path):
        """Test model save/load functionality"""
        model, X_test, y_test = trained_model
        
        # Save model
        model_path = tmp_path / "test_model"
        model.save_model(model_path)
        
        # Load model
        new_model = SPARCClassificationModel(model.config)
        new_model.load_model(model_path)
        
        # Test predictions are identical
        original_predictions = model.predict(X_test)
        loaded_predictions = new_model.predict(X_test)
        
        np.testing.assert_array_equal(original_predictions, loaded_predictions,
                                    "Loaded model predictions differ from original")
```

### Implementation Checklist

#### MLOps Foundation Setup (Phase 1)

✅ **Development Environment**
- [ ] Python environment with SPARC ML project structure
- [ ] DVC configured for data versioning
- [ ] MLflow setup for experiment tracking
- [ ] Git hooks for code quality checks
- [ ] Pre-commit hooks for data validation

✅ **Model Development Standards**
- [ ] Base model interface implemented following SPARC principles
- [ ] Model training pipeline with comprehensive logging
- [ ] Model evaluation framework with business metrics
- [ ] Model serialization and versioning system
- [ ] Feature engineering modules under 500 lines each

#### Production Deployment (Phase 2)

✅ **Model Serving Infrastructure**
- [ ] FastAPI serving application with monitoring
- [ ] Kubernetes deployment configurations
- [ ] Container images with model artifacts
- [ ] Load balancing and auto-scaling setup
- [ ] Health checks and readiness probes

✅ **Monitoring and Observability**
- [ ] Prometheus metrics collection
- [ ] Grafana dashboards for ML metrics
- [ ] Alerting rules for model degradation
- [ ] Log aggregation and analysis
- [ ] Performance monitoring and SLA tracking

#### Quality Assurance (Phase 3)

✅ **Testing Framework**
- [ ] Unit tests for all ML components
- [ ] Integration tests for end-to-end pipelines
- [ ] Model performance tests with thresholds
- [ ] Data quality validation tests
- [ ] Security and vulnerability testing

✅ **CI/CD Pipeline**
- [ ] Automated testing on code changes
- [ ] Model training and validation in CI
- [ ] Security scanning and compliance checks
- [ ] Automated deployment to staging/production
- [ ] Rollback procedures for failed deployments

This comprehensive MLOps standards document ensures that your SPARC ML engineering follows industry best practices while maintaining the modular, testable, and scalable principles core to SPARC methodology.
