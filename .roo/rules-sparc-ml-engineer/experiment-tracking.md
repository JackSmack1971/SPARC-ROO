        
        # Compare reproducibility hashes
        original_hash = original_results.get("reproducibility_hash", "")
        reproduced_hash = reproduced_results.get("reproducibility_hash", "")
        
        verification_results["hash_comparison"] = {
            "original_hash": original_hash,
            "reproduced_hash": reproduced_hash,
            "hashes_match": original_hash == reproduced_hash
        }
        
        # Determine overall reproducibility
        verification_results["reproducible"] = all_metrics_match and (original_hash == reproduced_hash)
        verification_results["verification_status"] = "passed" if verification_results["reproducible"] else "failed"
        
        # Generate recommendations
        if not verification_results["reproducible"]:
            recommendations = []
            if not all_metrics_match:
                recommendations.append("Metrics do not match within tolerance. Check random seed and data ordering.")
            if original_hash != reproduced_hash:
                recommendations.append("Reproducibility hashes differ. Environment or dependency versions may have changed.")
            verification_results["recommendations"] = recommendations
        
        # Update experiment results
        if self.current_experiment and self.current_experiment.experiment_id == reproduced_experiment_id:
            self.results.verification_status = verification_results["verification_status"]
        
        # Save verification results
        verification_path = self.experiments_dir / f"verification_{original_experiment_id}_{reproduced_experiment_id}.json"
        with open(verification_path, 'w') as f:
            json.dump(verification_results, f, indent=2)
        
        logger.info(f"Reproducibility verification {'passed' if verification_results['reproducible'] else 'failed'}")
        
        return verification_results
    
    def _capture_environment(self, config: ExperimentConfiguration) -> ExperimentConfiguration:
        """Capture complete environment information"""
        import sys
        import platform
        import pkg_resources
        
        # Python version
        config.python_version = sys.version
        
        # Dependencies
        dependencies = {}
        for package in pkg_resources.working_set:
            dependencies[package.project_name] = package.version
        config.dependencies = dependencies
        
        # Hardware configuration
        import psutil
        config.hardware_config = {
            "cpu_count": psutil.cpu_count(),
            "memory_total_gb": psutil.virtual_memory().total / (1024**3),
            "platform": platform.platform(),
            "processor": platform.processor(),
            "python_implementation": platform.python_implementation()
        }
        
        # GPU information if available
        try:
            import torch
            if torch.cuda.is_available():
                config.hardware_config["gpu_count"] = torch.cuda.device_count()
                config.hardware_config["gpu_name"] = torch.cuda.get_device_name(0)
        except ImportError:
            pass
        
        return config
    
    def _capture_git_state(self, config: ExperimentConfiguration) -> ExperimentConfiguration:
        """Capture Git repository state"""
        try:
            import git
            repo = git.Repo(search_parent_directories=True)
            
            config.git_commit_hash = repo.head.object.hexsha
            config.git_branch = repo.active_branch.name
            config.git_remote = repo.remotes.origin.url
            config.git_dirty = repo.is_dirty()
            
        except Exception as e:
            logger.warning(f"Could not capture Git state: {e}")
        
        return config
    
    def _calculate_file_hash(self, file_path: str) -> str:
        """Calculate SHA-256 hash of a file"""
        import hashlib
        
        sha256_hash = hashlib.sha256()
        with open(file_path, "rb") as f:
            for chunk in iter(lambda: f.read(4096), b""):
                sha256_hash.update(chunk)
        return sha256_hash.hexdigest()
    
    def _calculate_reproducibility_hash(self) -> str:
        """Calculate hash for reproducibility verification"""
        import hashlib
        
        # Combine key elements that affect reproducibility
        reproducibility_data = {
            "model_parameters": self.current_experiment.model_parameters,
            "training_parameters": self.current_experiment.training_parameters,
            "random_seed": self.current_experiment.random_seed,
            "dataset_hash": self.current_experiment.dataset_hash,
            "preprocessing_steps": self.current_experiment.preprocessing_steps,
            "python_version": self.current_experiment.python_version.split()[0],  # Major.minor version
            "key_dependencies": {
                k: v for k, v in self.current_experiment.dependencies.items()
                if k in ["scikit-learn", "pandas", "numpy", "tensorflow", "torch"]
            }
        }
        
        # Serialize and hash
        serialized = json.dumps(reproducibility_data, sort_keys=True)
        return hashlib.sha256(serialized.encode()).hexdigest()
    
    def _get_current_user(self) -> str:
        """Get current user name"""
        import getpass
        return getpass.getuser()
    
    def _serialize_config(self, config: ExperimentConfiguration) -> Dict[str, Any]:
        """Serialize experiment configuration to JSON-compatible format"""
        return {
            "experiment_id": config.experiment_id,
            "experiment_name": config.experiment_name,
            "created_at": config.created_at.isoformat(),
            "created_by": config.created_by,
            "hypothesis": {
                "problem_statement": config.hypothesis.problem_statement,
                "business_question": config.hypothesis.business_question,
                "success_criteria": config.hypothesis.success_criteria,
                "hypothesis_statement": config.hypothesis.hypothesis_statement,
                "expected_outcome": config.hypothesis.expected_outcome,
                "metrics_to_improve": config.hypothesis.metrics_to_improve,
                "baseline_metrics": config.hypothesis.baseline_metrics,
                "minimum_improvement": config.hypothesis.minimum_improvement,
                "statistical_significance_threshold": config.hypothesis.statistical_significance_threshold,
                "practical_significance_threshold": config.hypothesis.practical_significance_threshold
            },
            "experiment_type": config.experiment_type,
            "parent_experiment_id": config.parent_experiment_id,
            "related_experiments": config.related_experiments,
            "dataset_version": config.dataset_version,
            "dataset_hash": config.dataset_hash,
            "data_splits": config.data_splits,
            "feature_set": config.feature_set,
            "preprocessing_steps": config.preprocessing_steps,
            "model_type": config.model_type,
            "model_parameters": config.model_parameters,
            "training_parameters": config.training_parameters,
            "python_version": config.python_version,
            "dependencies": config.dependencies,
            "hardware_config": config.hardware_config,
            "random_seed": config.random_seed,
            "status": config.status,
            "start_time": config.start_time.isoformat() if config.start_time else None,
            "end_time": config.end_time.isoformat() if config.end_time else None,
            "execution_duration": config.execution_duration,
            "git_commit_hash": config.git_commit_hash,
            "git_branch": config.git_branch,
            "git_remote": config.git_remote,
            "git_dirty": config.git_dirty
        }
    
    def _serialize_results(self, results: ExperimentResults) -> Dict[str, Any]:
        """Serialize experiment results to JSON-compatible format"""
        return {
            "training_metrics": results.training_metrics,
            "validation_metrics": results.validation_metrics,
            "test_metrics": results.test_metrics,
            "cross_validation_metrics": results.cross_validation_metrics,
            "model_path": results.model_path,
            "model_size_mb": results.model_size_mb,
            "model_hash": results.model_hash,
            "predictions_path": results.predictions_path,
            "feature_importance": results.feature_importance,
            "confusion_matrix": results.confusion_matrix,
            "training_time_seconds": results.training_time_seconds,
            "memory_usage_peak_mb": results.memory_usage_peak_mb,
            "cpu_usage_average": results.cpu_usage_average,
            "gpu_usage_average": results.gpu_usage_average,
            "business_impact": results.business_impact,
            "cost_analysis": results.cost_analysis,
            "data_quality_score": results.data_quality_score,
            "model_quality_score": results.model_quality_score,
            "fairness_metrics": results.fairness_metrics,
            "statistical_significance": results.statistical_significance,
            "confidence_intervals": results.confidence_intervals,
            "plots_directory": results.plots_directory,
            "logs_directory": results.logs_directory,
            "artifacts_directory": results.artifacts_directory,
            "errors": results.errors,
            "warnings": results.warnings,
            "reproducibility_hash": results.reproducibility_hash,
            "verification_status": results.verification_status
        }
    
    def _load_experiment_results(self, experiment_id: str) -> Optional[Dict[str, Any]]:
        """Load experiment results from storage"""
        exp_dir = self.experiments_dir / experiment_id
        results_path = exp_dir / "results.json"
        
        if not results_path.exists():
            return None
        
        with open(results_path, 'r') as f:
            return json.load(f)
    
    def _create_experiment_summary(self) -> str:
        """Create markdown summary of experiment"""
        config = self.current_experiment
        results = self.results
        
        summary = f"""# Experiment Summary: {config.experiment_name}

## Experiment Information
- **Experiment ID**: {config.experiment_id}
- **Type**: {config.experiment_type}
- **Created By**: {config.created_by}
- **Duration**: {config.execution_duration:.2f} seconds
- **Status**: {config.status}

## Hypothesis
- **Problem Statement**: {config.hypothesis.problem_statement}
- **Business Question**: {config.hypothesis.business_question}
- **Hypothesis**: {config.hypothesis.hypothesis_statement}
- **Expected Outcome**: {config.hypothesis.expected_outcome}

## Model Configuration
- **Model Type**: {config.model_type}
- **Model Size**: {results.model_size_mb:.2f} MB
- **Training Time**: {results.training_time_seconds:.2f} seconds

## Performance Results
"""
        
        # Add validation metrics
        if results.validation_metrics:
            summary += "\n### Validation Metrics\n"
            for metric, value in results.validation_metrics.items():
                summary += f"- **{metric}**: {value:.4f}\n"
        
        # Add test metrics
        if results.test_metrics:
            summary += "\n### Test Metrics\n"
            for metric, value in results.test_metrics.items():
                summary += f"- **{metric}**: {value:.4f}\n"
        
        # Add business impact
        if results.business_impact:
            summary += "\n### Business Impact\n"
            for metric, value in results.business_impact.items():
                summary += f"- **{metric}**: {value:.4f}\n"
        
        # Add resource usage
        summary += f"""
## Resource Usage
- **Peak Memory**: {results.memory_usage_peak_mb:.2f} MB
- **Average CPU**: {results.cpu_usage_average:.2f}%
- **Average GPU**: {results.gpu_usage_average:.2f}%

## Reproducibility
- **Reproducibility Hash**: {results.reproducibility_hash}
- **Verification Status**: {results.verification_status}
- **Git Commit**: {config.git_commit_hash}
- **Git Branch**: {config.git_branch}
- **Random Seed**: {config.random_seed}

## Environment
- **Python Version**: {config.python_version.split()[0]}
- **Platform**: {config.hardware_config.get('platform', 'Unknown')}
- **CPU Count**: {config.hardware_config.get('cpu_count', 'Unknown')}
- **Memory**: {config.hardware_config.get('memory_total_gb', 'Unknown'):.2f} GB
"""
        
        return summary
    
    def _generate_comparison_recommendations(self, comparison_results: Dict[str, Any]) -> List[str]:
        """Generate recommendations based on experiment comparison"""
        recommendations = []
        
        # Find consistently best experiments
        best_counts = {}
        for metric, best_info in comparison_results["best_experiment"].items():
            exp_id = best_info["experiment_id"]
            best_counts[exp_id] = best_counts.get(exp_id, 0) + 1
        
        if best_counts:
            overall_best = max(best_counts.keys(), key=lambda k: best_counts[k])
            recommendations.append(f"Experiment {overall_best} performed best across most metrics")
        
        # Check for significant differences
        for metric, significance_tests in comparison_results["statistical_significance"].items():
            significant_differences = [
                test for test, results in significance_tests.items()
                if results["significant"]
            ]
            if significant_differences:
                recommendations.append(f"Significant differences found in {metric}: {', '.join(significant_differences)}")
        
        return recommendations
```

### Experiment Management and Organization

#### Advanced Experiment Organization

**Experiment Registry and Search**:
```python
# src/tracking/experiment_registry.py
from typing import Dict, List, Any, Optional
import pandas as pd
import sqlite3
from pathlib import Path
import json
import logging

logger = logging.getLogger(__name__)

class SPARCExperimentRegistry:
    """
    SPARC Experiment Registry
    
    Central registry for organizing, searching, and managing experiments
    across projects and teams. Provides advanced querying and analysis capabilities.
    """
    
    def __init__(self, registry_path: str = "experiment_registry.db"):
        self.registry_path = registry_path
        self.db_connection = sqlite3.connect(registry_path)
        self._initialize_database()
    
    def _initialize_database(self):
        """Initialize experiment registry database"""
        cursor = self.db_connection.cursor()
        
        # Create experiments table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS experiments (
                experiment_id TEXT PRIMARY KEY,
                experiment_name TEXT NOT NULL,
                experiment_type TEXT,
                status TEXT,
                created_by TEXT,
                created_at TIMESTAMP,
                end_time TIMESTAMP,
                execution_duration REAL,
                project_name TEXT,
                git_commit_hash TEXT,
                git_branch TEXT,
                model_type TEXT,
                dataset_hash TEXT,
                reproducibility_hash TEXT,
                hypothesis_statement TEXT,
                parent_experiment_id TEXT,
                tags TEXT  -- JSON string of tags
            )
        """)
        
        # Create metrics table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS experiment_metrics (
                metric_id INTEGER PRIMARY KEY AUTOINCREMENT,
                experiment_id TEXT,
                metric_name TEXT,
                metric_value REAL,
                metric_type TEXT,  -- 'training', 'validation', 'test', 'business'
                FOREIGN KEY (experiment_id) REFERENCES experiments (experiment_id)
            )
        """)
        
        # Create parameters table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS experiment_parameters (
                param_id INTEGER PRIMARY KEY AUTOINCREMENT,
                experiment_id TEXT,
                param_name TEXT,
                param_value TEXT,  -- JSON string for complex values
                param_type TEXT,   -- 'model', 'training', 'data'
                FOREIGN KEY (experiment_id) REFERENCES experiments (experiment_id)
            )
        """)
        
        # Create indexes for efficient querying
        cursor.execute("CREATE INDEX IF NOT EXISTS idx_experiment_type ON experiments (experiment_type)")
        cursor.execute("CREATE INDEX IF NOT EXISTS idx_created_by ON experiments (created_by)")
        cursor.execute("CREATE INDEX IF NOT EXISTS idx_model_type ON experiments (model_type)")
        cursor.execute("CREATE INDEX IF NOT EXISTS idx_metric_name ON experiment_metrics (metric_name)")
        cursor.execute("CREATE INDEX IF NOT EXISTS idx_param_name ON experiment_parameters (param_name)")
        
        self.db_connection.commit()
    
    def register_experiment(self, experiment_config: ExperimentConfiguration,
                          experiment_results: ExperimentResults,
                          project_name: str = None,
                          tags: List[str] = None) -> None:
        """
        Register completed experiment in the registry
        """
        logger.info(f"Registering experiment {experiment_config.experiment_id}")
        
        cursor = self.db_connection.cursor()
        
        # Insert experiment record
        cursor.execute("""
            INSERT OR REPLACE INTO experiments 
            (experiment_id, experiment_name, experiment_type, status, created_by,
             created_at, end_time, execution_duration, project_name, git_commit_hash,
             git_branch, model_type, dataset_hash, reproducibility_hash,
             hypothesis_statement, parent_experiment_id, tags)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
        """, (
            experiment_config.experiment_id,
            experiment_config.experiment_name,
            experiment_config.experiment_type,
            experiment_config.status,
            experiment_config.created_by,
            experiment_config.created_at,
            experiment_config.end_time,
            experiment_config.execution_duration,
            project_name,
            experiment_config.git_commit_hash,
            experiment_config.git_branch,
            experiment_config.model_type,
            experiment_config.dataset_hash,
            experiment_results.reproducibility_hash,
            experiment_config.hypothesis.hypothesis_statement if experiment_config.hypothesis else None,
            experiment_config.parent_experiment_id,
            json.dumps(tags) if tags else None
        ))
        
        # Insert metrics
        all_metrics = {
            **{f"training_{k}": v for k, v in experiment_results.training_metrics.items()},
            **{f"validation_{k}": v for k, v in experiment_results.validation_metrics.items()},
            **{f"test_{k}": v for k, v in experiment_results.test_metrics.items()},
            **{f"business_{k}": v for k, v in experiment_results.business_impact.items()}
        }
        
        for metric_name, metric_value in all_metrics.items():
            metric_type = metric_name.split('_')[0]  # Extract type prefix
            cursor.execute("""
                INSERT INTO experiment_metrics 
                (experiment_id, metric_name, metric_value, metric_type)
                VALUES (?, ?, ?, ?)
            """, (experiment_config.experiment_id, metric_name, metric_value, metric_type))
        
        # Insert parameters
        all_parameters = {
            **{f"model_{k}": v for k, v in experiment_config.model_parameters.items()},
            **{f"training_{k}": v for k, v in experiment_config.training_parameters.items()},
            "data_dataset_hash": experiment_config.dataset_hash,
            "data_dataset_version": experiment_config.dataset_version
        }
        
        for param_name, param_value in all_parameters.items():
            param_type = param_name.split('_')[0]  # Extract type prefix
            cursor.execute("""
                INSERT INTO experiment_parameters 
                (experiment_id, param_name, param_value, param_type)
                VALUES (?, ?, ?, ?)
            """, (experiment_config.experiment_id, param_name, json.dumps(param_value), param_type))
        
        self.db_connection.commit()
        logger.info(f"Experiment {experiment_config.experiment_id} registered successfully")
    
    def search_experiments(self, 
                          experiment_type: str = None,
                          model_type: str = None,
                          created_by: str = None,
                          project_name: str = None,
                          min_metric_value: Dict[str, float] = None,
                          max_metric_value: Dict[str, float] = None,
                          tags: List[str] = None,
                          limit: int = 100) -> pd.DataFrame:
        """
        Search experiments based on various criteria
        """
        query = """
            SELECT e.*, 
                   GROUP_CONCAT(m.metric_name || ':' || m.metric_value) as metrics
            FROM experiments e
            LEFT JOIN experiment_metrics m ON e.experiment_id = m.experiment_id
            WHERE 1=1
        """
        params = []
        
        # Add filters
        if experiment_type:
            query += " AND e.experiment_type = ?"
            params.append(experiment_type)
        
        if model_type:
            query += " AND e.model_type = ?"
            params.append(model_type)
        
        if created_by:
            query += " AND e.created_by = ?"
            params.append(created_by)
        
        if project_name:
            query += " AND e.project_name = ?"
            params.append(project_name)
        
        # Add metric filters
        if min_metric_value:
            for metric_name, min_value in min_metric_value.items():
                query += f"""
                    AND e.experiment_id IN (
                        SELECT experiment_id FROM experiment_metrics 
                        WHERE metric_name = ? AND metric_value >= ?
                    )
                """
                params.extend([metric_name, min_value])
        
        if max_metric_value:
            for metric_name, max_value in max_metric_value.items():
                query += f"""
                    AND e.experiment_id IN (
                        SELECT experiment_id FROM experiment_metrics 
                        WHERE metric_name = ? AND metric_value <= ?
                    )
                """
                params.extend([metric_name, max_value])
        
        # Add tag filters
        if tags:
            for tag in tags:
                query += " AND (e.tags LIKE ? OR e.tags LIKE ? OR e.tags LIKE ?)"
                params.extend([f'%"{tag}"%', f'%[{tag},%', f'%, {tag}%'])
        
        query += f" GROUP BY e.experiment_id ORDER BY e.created_at DESC LIMIT ?"
        params.append(limit)
        
        return pd.read_sql_query(query, self.db_connection, params=params)
    
    def get_experiment_leaderboard(self, metric_name: str, 
                                 experiment_type: str = None,
                                 limit: int = 10) -> pd.DataFrame:
        """
        Get top experiments ranked by specific metric
        """
        query = """
            SELECT e.experiment_id, e.experiment_name, e.model_type, e.created_by,
                   m.metric_value, e.created_at
            FROM experiments e
            JOIN experiment_metrics m ON e.experiment_id = m.experiment_id
            WHERE m.metric_name = ?
        """
        params = [metric_name]
        
        if experiment_type:
            query += " AND e.experiment_type = ?"
            params.append(experiment_type)
        
        query += " ORDER BY m.metric_value DESC LIMIT ?"
        params.append(limit)
        
        return pd.read_sql_query(query, self.db_connection, params=params)
    
    def analyze_experiment_trends(self, metric_name: str,
                                group_by: str = "model_type") -> pd.DataFrame:
        """
        Analyze trends in experiment performance over time
        """
        query = f"""
            SELECT e.{group_by}, 
                   AVG(m.metric_value) as avg_metric,
                   MAX(m.metric_value) as max_metric,
                   MIN(m.metric_value) as min_metric,
                   COUNT(*) as experiment_count,
                   DATE(e.created_at) as date
            FROM experiments e
            JOIN experiment_metrics m ON e.experiment_id = m.experiment_id
            WHERE m.metric_name = ?
            GROUP BY e.{group_by}, DATE(e.created_at)
            ORDER BY date DESC
        """
        
        return pd.read_sql_query(query, self.db_connection, params=[metric_name])
```

### Implementation Checklist

#### Experiment Tracking Setup (Phase 1)

✅ **Core Tracking Infrastructure**
- [ ] Implement SPARCExperimentTracker with MLflow integration
- [ ] Set up experiment schema and data models
- [ ] Configure automatic environment and Git state capture
- [ ] Implement comprehensive metadata tracking
- [ ] Create experiment directory structure automation

✅ **Reproducibility Framework**
- [ ] Implement reproducibility hash calculation
- [ ] Create experiment reproduction capabilities
- [ ] Set up verification and validation workflows
- [ ] Implement dependency and environment tracking
- [ ] Create reproducibility testing automation

#### Advanced Features (Phase 2)

✅ **Experiment Registry and Search**
- [ ] Implement centralized experiment registry
- [ ] Create advanced search and filtering capabilities
- [ ] Set up experiment leaderboards and rankings
- [ ] Implement trend analysis and reporting
- [ ] Create experiment comparison and visualization tools

✅ **Team Collaboration**
- [ ] Set up shared experiment tracking infrastructure
- [ ] Implement experiment sharing and review workflows
- [ ] Create team dashboards and reporting
- [ ] Set up experiment approval and governance processes
- [ ] Implement knowledge sharing and documentation automation

#### Quality Assurance (Phase 3)

✅ **Validation and Testing**
- [ ] Implement experiment validation checks
- [ ] Create automated quality gates for experiments
- [ ] Set up experiment review and approval workflows
- [ ] Implement statistical significance testing
- [ ] Create experiment audit and compliance tracking

✅ **Integration and Automation**
- [ ] Integrate with CI/CD pipelines
- [ ] Set up automated experiment triggers
- [ ] Implement model deployment integration
- [ ] Create monitoring and alerting for experiments
- [ ] Set up automated reporting and notifications

This comprehensive experiment tracking framework ensures that every SPARC ML experiment is fully traceable, reproducible, and contributes to organizational learning and model improvement over time.
