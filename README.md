# 🚀 Kafka Performance Tester on OpenShift (with CPU Tuning)

This project helps test how changing CPU power settings (C-State tuning) impacts Kafka producer and consumer throughput in an OpenShift environment.

---

## 🎯 What's This For?

Kafka performance isn't just about Kafka itself—it depends on:
- 🧠 **CPU wake-up time** (affected by C-States)
- 💽 **Disk IOPS** (read/write speed matters!)
- 🌐 **Network** (moving messages around)

By tweaking CPU settings, we can compare how fast Kafka runs in:
- **Normal mode** (`max_cstate=9`)  
- **Performance mode** (`max_cstate=1`)

---

## 🧠 Kafka Performance Flow (with C-State & Disk Impact)

```mermaid
flowchart TD
    Producer[Producer App - Sends Messages] -->|Write| Topic[Kafka Topic - test-topic]
    Consumer[Consumer App - Reads Messages] -->|Read| Topic

    Topic --> Broker[Kafka Broker]
    Broker --> Disk[Disk IOPS]
    Broker --> CPU[CPU Core]
    CPU --> CState[CPU C-State: max_cstate=1 or 9]

    subgraph Performance_Impact
        CState -->|High = Slow Wakeup| Latency1[Kafka Slower]
        CState -->|Low = Fast Wakeup| Latency2[Kafka Faster]
        Disk -->|Slow Disk = IO bottleneck| IO1[Lower Throughput]
        Disk -->|Fast Disk = Smooth writes| IO2[Higher Throughput]
    end
````

---

## ⚙️ What You'll Need

* ✅ OpenShift 4.x cluster (admin access)
* ✅ Worker nodes that allow CPU tuning
* ✅ `oc` CLI configured
* ✅ Internet inside pods (for Python libs)

---

## 🚀 How to Use

### 1. Clone This Repo

```bash
git clone https://github.com/BabbarPB08/kafka-perf-tester.git
cd kafka-perf-tester/
```

### 2. Create a Namespace

```bash
oc apply -k kafka-strimzy/ns/
```

Expected:

```
project.project.openshift.io/kafka created
```

### 3. Deploy Kafka & Test Setup

```bash
oc apply -k kafka-strimzy/resources/
```

This creates:

* Kafka cluster (via Strimzi)
* Kafka topic (`test-topic`)
* Test script ConfigMap
* Kafka test client pod

### 4. Check if Pods are Ready

```bash
oc get pods -n kafka
```

Sample output:

```
kafka-test-client-xxxxx             1/1     Running
my-cluster-kafka-0                 1/1     Running
...
```

---

## 🔍 Run Benchmark

### 🧪 Before CPU Tuning

```bash
oc rsh -n kafka kafka-test-client-xxxxx
```

Inside the pod:

```bash
cat /sys/module/intel_idle/parameters/max_cstate  # should return 9
pip install confluent-kafka
python /opt/scripts/kafka_perf_test.py
```

Sample output:

```
==== SUMMARY ====
Avg Producer Time: 24.60s
Producer Throughput: 40656 msgs/sec
Avg Consumer Time: 12.30s
Consumer Throughput: 81282 msgs/sec
```

---

## 🔧 Tune the CPU (Set C-State)

Follow Red Hat guide 👉 [KCS 7123874](https://access.redhat.com/solutions/7123874):

1. Apply kernel arg via MachineConfig:

```bash
intel_idle.max_cstate=1
```

2. Reboot worker nodes to apply changes.

---

## 🧪 Run Benchmark Again

Inside test client pod:

```bash
cat /sys/module/intel_idle/parameters/max_cstate  # should return 1 now
python /opt/scripts/kafka_perf_test.py
```

Sample output:

```
==== SUMMARY ====
Avg Producer Time: 26.26s
Producer Throughput: 38078 msgs/sec
Avg Consumer Time: 7.65s
Consumer Throughput: 130693 msgs/sec
```

---

## 📊 Result Comparison

| C-State        | Producer Time | Producer TPS  | Consumer Time | Consumer TPS   |
| -------------- | ------------- | ------------- | ------------- | -------------- |
| `max_cstate=9` | 24.60 sec     | 40,656 msgs/s | 12.30 sec     | 81,282 msgs/s  |
| `max_cstate=1` | 26.26 sec     | 38,078 msgs/s | 7.65 sec      | 130,693 msgs/s |

> 🟢 **Consumer got a big boost** after tuning
> 🔴 **Producer dropped a little**—likely due to CPU no longer saving energy

---

## 📌 Conclusion

* Kafka is sensitive to CPU wake-up times and disk speed
* Lowering `max_cstate` can improve **latency-sensitive** workloads
* You can use this repo to **test before & after** changes in OpenShift

---

## 📚 Reference

* 🧠 Red Hat C-State tuning guide: [KCS Solution 7123874](https://access.redhat.com/solutions/7123874)

---

## 🙌 Maintainer

Made with ❤️ by [@BabbarPB08](https://github.com/BabbarPB08)


