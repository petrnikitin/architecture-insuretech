# Task 2 - Часть 2: Масштабирование по RPS (кастомные метрики)

## Установленные компоненты

1. **Prometheus** - система мониторинга
2. **Prometheus Adapter** - адаптер для передачи кастомных метрик в Kubernetes
3. **ServiceMonitor** - конфигурация для сбора метрик с приложения

## Созданные файлы

- `servicemonitor.yaml` - ServiceMonitor для сбора метрик `/metrics` с приложения
- `prometheus-adapter-config.yaml` - конфигурация Adapter для метрики RPS
- `hpa-rps.yaml` - HPA для масштабирования по `http_requests_per_second` (target: 10 RPS/pod)

## Проверка

```bash
# Prometheus UI
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090

# Кастомные метрики
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1

# HPA
kubectl get hpa scaletestapp-hpa-rps
```

## Скриншоты

См. файлы в директории Task2.
