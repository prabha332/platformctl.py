# platformctl.py
#Single-file, Production-ready Kubernetes Platform Controller

#!/usr/bin/env python3

import os
import sys
import logging
import argparse
import subprocess
from datetime import datetime
from kubernetes import client, config
import requests

# ==============================
# Configuration
# ==============================

LOG_FILE = "platformctl.log"
SLACK_WEBHOOK = os.getenv("SLACK_WEBHOOK")
NAMESPACE = "default"

# ==============================
# Logging Setup
# ==============================

def setup_logging():
    logging.basicConfig(
        filename=LOG_FILE,
        level=logging.INFO,
        format="%(asctime)s - %(levelname)s - %(message)s"
    )
    logging.getLogger().addHandler(logging.StreamHandler(sys.stdout))


# ==============================
# Kubernetes Setup
# ==============================

def load_kube():
    try:
        config.load_kube_config()
        logging.info("Kubernetes config loaded.")
    except Exception as e:
        logging.error(f"Failed to load kubeconfig: {e}")
        sys.exit(1)


# ==============================
# Deploy Application
# ==============================

def deploy_app(manifest):
    try:
        subprocess.run(["kubectl", "apply", "-f", manifest], check=True)
        logging.info(f"Deployment applied from {manifest}")
    except subprocess.CalledProcessError as e:
        logging.error(f"Deployment failed: {e}")
        send_slack_alert(f"Deployment failed: {e}")


# ==============================
# Setup Horizontal Pod Autoscaler
# ==============================

def setup_hpa(deployment, min_pods, max_pods, cpu_percent):
    try:
        subprocess.run([
            "kubectl", "autoscale", "deployment", deployment,
            "--cpu-percent", str(cpu_percent),
            "--min", str(min_pods),
            "--max", str(max_pods)
        ], check=True)
        logging.info("HPA configured successfully.")
    except subprocess.CalledProcessError as e:
        logging.error(f"HPA setup failed: {e}")


# ==============================
# Install Monitoring Stack
# ==============================

def install_monitoring():
    try:
        subprocess.run([
            "helm", "install", "monitoring",
            "prometheus-community/kube-prometheus-stack"
        ], check=True)
        logging.info("Monitoring stack installed.")
    except subprocess.CalledProcessError as e:
        logging.error(f"Monitoring installation failed: {e}")


# ==============================
# Auto-Healing Logic
# ==============================

def auto_heal():
    v1 = client.CoreV1Api()
    pods = v1.list_namespaced_pod(NAMESPACE)

    for pod in pods.items:
        phase = pod.status.phase
        name = pod.metadata.name

        if phase != "Running":
            logging.warning(f"Pod {name} unhealthy. Deleting...")
            try:
                v1.delete_namespaced_pod(name, NAMESPACE)
                logging.info(f"Pod {name} deleted for auto-heal.")
                send_slack_alert(f"Auto-healed pod: {name}")
            except Exception as e:
                logging.error(f"Failed to delete pod {name}: {e}")


# ==============================
# Slack Alerts
# ==============================

def send_slack_alert(message):
    if not SLACK_WEBHOOK:
        logging.warning("Slack webhook not configured.")
        return

    payload = {"text": message}
    try:
        requests.post(SLACK_WEBHOOK, json=payload)
    except Exception as e:
        logging.error(f"Slack alert failed: {e}")


# ==============================
# Cluster Health Monitoring
# ==============================

def monitor_cluster():
    v1 = client.CoreV1Api()
    nodes = v1.list_node()

    for node in nodes.items:
        for condition in node.status.conditions:
            if condition.type == "Ready" and condition.status != "True":
                msg = f"Node {node.metadata.name} Not Ready!"
                logging.error(msg)
                send_slack_alert(msg)


# ==============================
# CLI Interface
# ==============================

def main():
    parser = argparse.ArgumentParser(description="PlatformCTL Kubernetes Controller")

    parser.add_argument("--deploy", help="Deploy app using manifest file")
    parser.add_argument("--hpa", nargs=4, metavar=("DEPLOYMENT", "MIN", "MAX", "CPU"))
    parser.add_argument("--monitor", action="store_true")
    parser.add_argument("--heal", action="store_true")
    parser.add_argument("--install-monitoring", action="store_true")

    args = parser.parse_args()

    setup_logging()
    load_kube()

    if args.deploy:
        deploy_app(args.deploy)

    if args.hpa:
        deployment, min_pods, max_pods, cpu = args.hpa
        setup_hpa(deployment, min_pods, max_pods, cpu)

    if args.install_monitoring:
        install_monitoring()

    if args.monitor:
        monitor_cluster()

    if args.heal:
        auto_heal()


if __name__ == "__main__":
    main()
