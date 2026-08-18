# Experiment 01 — Pod Kill (PodChaos)

## Steady State (before)
- podinfo running with 3 replicas, all 1/1 Ready
- loadgen sending ~5 req/s
- Grafana: sum(rate(http_requests_total{namespace="demo"}[1m])) ≈ 5, flat line

## Blast Radius
- Target: namespace=demo, app=podinfo, ONE pod only

## Hypothesis
- Killing 1 of 3 podinfo pods will NOT drop the success-rate line to zero.
- The remaining 2 replicas will absorb the traffic (brief dip at most).
- Kubernetes will detect the missing pod and create a new one (self-healing),
  returning the deployment to 3/3 within seconds.

## Actual Result (fill after run)
- Request rate during kill: ~5 req/s বজায় ছিল, শূন্যে নামেনি
- Did any requests fail?: সামান্য dip (একটা ছোট notch), কিন্তু কার্যত downtime নেই
- Time to recover to 3/3: কয়েক সেকেন্ড (নতুন pod AGE ~68s এ 3/3 confirmed)
- Screenshot(s): 02-podkill-selfhealing-podlist.png, 03-podkill-grafana-dip.png

## Conclusion
- Resilient ✅. 3 replica থাকায় single pod-kill ব্যবহারকারীর কাছে প্রায় অদৃশ্য।
- Kubernetes-এর desired-state reconciliation নিজে থেকে pod recreate করে 3/3 ফিরিয়ে আনে।
- Learning: resilience দেখানোর সবচেয়ে ছোট অর্থবহ replica সংখ্যা হলো 3।
